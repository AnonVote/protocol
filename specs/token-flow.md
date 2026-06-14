# Token Flow Specification

This document describes the end-to-end flow from eligibility list upload through vote submission and result publication.

---

## Overview

```
Admin                    Backend                     Voter
  │                         │                          │
  │── Upload CSV ──────────►│                          │
  │                         │ hashIdentifier(id)       │
  │                         │ store identifierHash     │
  │                         │                          │
  │                         │◄── Enter identifier ─────│
  │                         │                          │
  │                         │ hashIdentifier(input)    │
  │                         │ lookup in EligibilityEntry│
  │                         │ generateToken()          │
  │                         │ hashToken(rawToken)      │
  │                         │ store tokenHash          │
  │                         │ set tokenIssued = true   │
  │                         │── rawToken ─────────────►│
  │                         │                          │
  │                         │◄── rawToken + optionId ──│
  │                         │                          │
  │                         │ hashToken(rawToken)      │
  │                         │ lookup VoterToken        │
  │                         │ encryptVote(optionId)    │
  │                         │ store Vote               │
  │                         │ mark token used          │
  │                         │ write to Stellar         │
  │                         │── "Vote recorded" ──────►│
  │                         │                          │
  │── Tally ballot ────────►│                          │
  │                         │ decryptVote(payload) ×N  │
  │                         │ aggregate tally          │
  │                         │ store Result             │
  │                         │ write to Stellar         │
```

---

## Step-by-step

### 1. Eligibility list upload (admin)

- Admin uploads a CSV / newline-separated list of voter identifiers
- Backend calls `hashIdentifier(id)` for each entry
- Stores `{ eligibilityListId, identifierHash, weight, tokenIssued: false }` in `EligibilityEntry`
- **Raw identifiers are never written to the database**

### 2. Token request (voter)

- Voter submits their identifier to `POST /api/tokens`
- Backend calls `hashIdentifier(input)` and looks up the hash in `EligibilityEntry`
- If found and `tokenIssued = false`:
  - Calls `generateToken()` → `rawToken`
  - Calls `hashToken(rawToken)` → `tokenHash`
  - Stores `{ tokenHash, ballotId, used: false }` in `VoterToken`
  - Sets `EligibilityEntry.tokenIssued = true`
  - Returns `rawToken` to the voter
- **`rawToken` is returned once and never stored**

### 3. Vote submission (voter)

- Voter submits `rawToken` + `optionId` to `POST /api/votes`
- Backend calls `hashToken(rawToken)` and looks up the hash in `VoterToken`
- Validates: token exists, belongs to ballot, not already used
- Calls `encryptVote(optionId, BALLOT_ENCRYPTION_KEY)` → `encryptedPayload`
- Stores `{ ballotId, optionId, encryptedPayload, weight, rank? }` in `Vote`
- Marks `VoterToken.used = true`
- Writes `VOTE_CAST` audit event to Stellar (fire-and-forget)

### 4. Result tally (admin or scheduler)

- Admin calls `POST /api/results/:ballotId/tally` or scheduler auto-closes at deadline
- Backend loads all `Vote` records for the ballot
- Calls `decryptVote(payload, BALLOT_ENCRYPTION_KEY)` for each vote
- Aggregates `tally[optionId] += vote.weight`
- Checks consistency: `SUM(vote.weight) == COUNT(used tokens)`
- Stores `Result` with `tallyJson`, `totalVotes`, `isConsistent`
- Writes `RESULT_PUBLISHED` event to Stellar

---

## Token reissue flow

If a voter loses their token before voting:

1. Voter requests reissue at `POST /api/tokens/reissue`
2. Backend verifies `tokenIssued = true` but no corresponding used token exists
3. Backend deletes the old unused `VoterToken` and creates a new one
4. Returns a fresh `rawToken`

If the voter's token was already used to vote, reissue is blocked with a clear error.

---

## Delegation flow

1. Delegator calls `POST /api/delegations` with their token hash and the delegate's token hash
2. Backend sets `delegatorToken.delegatedTo = delegateToken.id`, marks delegator token used
3. At vote submission time, backend resolves the delegation chain and uses the delegate's token
