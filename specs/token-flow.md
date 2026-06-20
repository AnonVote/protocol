# Token Flow Specification

**Scope:** eligibility ingestion through token invalidation after vote
submission<br>
**Core routes:** `POST /api/eligibility`, `POST /api/tokens`,
`POST /api/tokens/reissue`, `POST /api/votes`,
`POST /api/tokens/reset/:ballotId`<br>
**Crypto references:** `hashIdentifier`, `generateToken`, `hashToken`,
`encryptVote` from `AnonVote/js/src/crypto.ts`<br>
**Core service references:** `identityManager.issueToken`,
`identityManager.reissueToken`, `privacyEngine.submitVote`,
`stellarService.writeRecord`, and the Soroban wrappers in `sorobanService`

This document is the normative AnonVote token lifecycle specification. It
describes how a raw voter identifier becomes an eligibility hash, how a raw
token is issued without being stored, how a raw token is redeemed exactly once,
and how replacement tokens are handled without linking a voter to a vote.

The privacy invariant is:

```text
EligibilityEntry.identifierHash MUST NOT be joinable to Vote.encryptedPayload.
VoterToken.tokenHash authorizes voting, but raw tokens MUST NOT be stored.
```

---

## Data Model Summary

The token flow relies on these persistent records:

| Model | Relevant fields | Privacy rule |
| --- | --- | --- |
| `EligibilityEntry` | `eligibilityListId`, `identifierHash`, `weight`, `tokenIssued` | Stores only `hashIdentifier(identifier)`; never store the raw identifier. |
| `VoterToken` | `tokenHash`, `ballotId`, `used`, `issuedAt`, `usedAt`, `delegatedFrom`, `delegatedTo` | Stores only `hashToken(rawToken)`; never store the raw token. |
| `Vote` | `ballotId`, `optionId`, `encryptedPayload`, `weight`, `rank`, `stellarTxId` | Stores encrypted option payload; never store the token or identifier. |
| `AuditEvent` | `ballotId`, `eventType`, `stellarTxId`, `stellarLedgerAt` | Stores event type and ballot context only; never store identifiers or tokens. |

`EligibilityEntry` has a unique constraint on
`(eligibilityListId, identifierHash)`. `VoterToken.tokenHash` is unique.

---

## Phase 1 - Eligibility Ingestion

### Entry Point

Core route: `POST /api/eligibility`

The administrator uploads a CSV or plain-text eligibility file. The request is
authenticated with the organization session cookie.

### Required Steps

1. Read the uploaded file as UTF-8.
2. Remove a UTF-8 BOM if present.
3. Split on line boundaries.
4. Trim each line, normalize internal whitespace where core requires it, and
   discard empty rows.
5. Reject the upload if it exceeds file size, row count, or per-row length
   limits.
6. For every sanitized identifier, compute:

   ```text
   identifierHash = hashIdentifier(identifier)
   ```

   `hashIdentifier` trims and lowercases before SHA-256 hashing.

7. Deduplicate by `identifierHash` within the same upload.
8. In a database transaction:
   - create one `EligibilityList`
   - create one `EligibilityEntry` per unique hash with
     `tokenIssued = false`
9. Return the `eligibilityListId` and accepted entry count.

### Stored vs Discarded Data

Stored:

- `EligibilityList.id`
- `EligibilityEntry.eligibilityListId`
- `EligibilityEntry.identifierHash`
- optional voter `weight`
- `EligibilityEntry.tokenIssued = false`

Discarded:

- raw identifier strings
- CSV row order
- casing and surrounding whitespace differences
- malformed or empty entries

### Duplicate Handling

Duplicates MUST be detected after hashing. This ensures
`" Alice@example.com "` and `"alice@example.com"` map to one eligibility entry.
Within one eligibility list, duplicate hashes MUST NOT create multiple voting
rights.

If duplicates appear in a file, the accepted count SHOULD report unique voters,
not raw rows. If an existing list is updated in a future implementation, the
same uniqueness rule MUST apply across the existing and new entries.

---

## Phase 2 - Token Issuance

### Entry Point

Core route: `POST /api/tokens`

Request body:

```json
{
  "ballotId": "uuid",
  "voterIdentifier": "alice@example.com"
}
```

The endpoint is public because voters do not yet have an AnonVote credential.
It MUST be rate limited. Core currently applies `strictRateLimiter`.

### Required Steps

1. Require `ballotId` and `voterIdentifier`.
2. Trim `voterIdentifier` before service-level processing.
3. Load the ballot and its `eligibilityListId`.
4. Reject closed or unknown ballots with a generic error that does not reveal
   whether the identifier is eligible.
5. Compute:

   ```text
   identifierHash = hashIdentifier(voterIdentifier)
   ```

6. Look up `EligibilityEntry` by `(eligibilityListId, identifierHash)`.
7. If the entry is missing, reject with a generic eligibility error.
8. If `tokenIssued = true`, record `DUPLICATE_TOKEN_ATTEMPT` and return a
   `TokenAlreadyIssued` style error unless the system can prove the voter has
   already voted, in which case return an `AlreadyVoted` style error.
9. Generate and hash the token in sequence:

   ```text
   rawToken  = generateToken()
   tokenHash = hashToken(rawToken)
   ```

10. In a database transaction:
    - create `VoterToken { tokenHash, ballotId, used: false }`
    - update the eligibility entry to `tokenIssued = true`
    - create `AuditEvent { ballotId, eventType: "TOKEN_ISSUED" }`
11. Deliver `rawToken` to the voter exactly once.
12. Write the token-issued audit event to the active ledger layer:
    - current core: `stellarService.writeRecord({ type: "TOKEN_ISSUED", ... })`
    - Soroban-ready wrapper: `sorobanRecordToken(ballotIdHash)`

### Stored vs Delivered Data

Stored server-side:

- `hashToken(rawToken)` as `VoterToken.tokenHash`
- `ballotId`
- `used = false`
- `issuedAt`
- audit event metadata

Delivered to voter:

- `rawToken`
- voter `weight` if weighted voting is enabled

Never stored:

- `rawToken`
- `voterIdentifier`
- email delivery contents containing the token

### Delivery Requirements

The production delivery channel MAY be email, an authenticated admin-mediated
channel, or the current core API response consumed by the voter UI. Regardless
of channel, the requirements are identical:

1. The raw token MUST be shown or sent only once.
2. The raw token MUST NOT be logged by route handlers, mail providers, error
   tracking, analytics, or request tracing.
3. If email delivery is used, the system SHOULD record delivery attempt metadata
   separately from token value:
   - `ballotId`
   - `AuditEvent` or delivery event ID
   - provider message ID
   - timestamp
   - success/failure status
4. Delivery confirmation MUST NOT include the raw token. It may reference the
   audit event ID or provider message ID.
5. If delivery fails before the voter can receive the token, the token issuance
   transaction MUST either be rolled back or the failed token MUST be invalidated
   before any replacement token is issued.

### Failure Modes

- Missing `ballotId` or `voterIdentifier`: reject before hashing.
- Unknown ballot, closed ballot, or ineligible identifier: reject without
  revealing more state than necessary.
- Duplicate token request: record `DUPLICATE_TOKEN_ATTEMPT` without storing the
  identifier.
- Database unique collision on `tokenHash`: retry with a fresh token or abort
  before returning a raw token.
- Ledger write failure: MUST NOT expose the raw token in logs. If the ledger
  write is asynchronous, the audit event should remain in database state and be
  retriable.

---

## Phase 3 - Token Redemption

### Entry Point

Core route: `POST /api/votes`

Request body:

```json
{
  "ballotId": "uuid",
  "voterToken": "64-char-hex-string",
  "optionId": "uuid",
  "weight": 1,
  "rank": null
}
```

The endpoint MUST be rate limited. Core currently applies `strictRateLimiter`.

### Required Steps

1. Require `ballotId`, `voterToken`, and `optionId`.
2. Trim `voterToken`.
3. Compute:

   ```text
   tokenHash = hashToken(voterToken)
   ```

4. Resolve delegation if enabled. Current core calls
   `delegationManager.getEffectiveVoter(ballotId, tokenHash)` before final
   token lookup.
5. Look up `VoterToken` by token hash. Validation MUST be hash comparison, not
   plaintext comparison.
6. Reject if the token is missing or belongs to another ballot.
7. Reject if `VoterToken.used = true`. Record `DUPLICATE_VOTE_ATTEMPT` without
   storing the raw token.
8. Load the ballot and options.
9. Reject if the ballot is missing, closed, or past its voting deadline.
10. Reject if `optionId` is not an option in the ballot.
11. Encrypt the vote:

    ```text
    encryptedPayload = encryptVote(optionId, ballotKey)
    ```

12. Execute the redemption transaction described below.
13. Write the vote-cast audit event to the active ledger layer:
    - current core: `stellarService.writeRecord({ type: "VOTE_CAST", ... })`
    - Soroban-ready wrapper: `sorobanRecordVote(ballotIdHash)`
14. Return vote confirmation without returning token data.

### Atomicity Requirement

The following operations MUST be one database transaction:

1. Create `Vote { ballotId, optionId, encryptedPayload, weight, rank }`.
2. Update the effective `VoterToken` to `used = true` and set `usedAt`.
3. Create `AuditEvent { ballotId, eventType: "VOTE_CAST" }`.

This transaction is the boundary that prevents double voting. A token is valid
only if the vote write and token invalidation commit together.

The transaction MUST NOT include external network calls such as Stellar,
Soroban, email, or webhook delivery. External writes can be retried using the
database `AuditEvent` or `Vote` ID after the core transaction commits.

### Mid-Transaction Failure

If the transaction fails before commit:

- no `Vote` row may persist
- `VoterToken.used` must remain `false`
- no `VOTE_CAST` audit event may persist
- the voter may retry with the same token

If the database transaction commits but the ledger write fails:

- the vote remains valid
- the token remains invalidated
- the audit event remains stored without `stellarTxId`
- a retry worker may later write the audit event to Stellar or Soroban

### Failure Modes

- Invalid token: reject with a generic invalid token error.
- Used token: reject and record `DUPLICATE_VOTE_ATTEMPT`.
- Expired or closed ballot: reject before creating the vote.
- Invalid option: reject before encryption and before token invalidation.
- Encryption failure: abort before the transaction; the token remains unused.

---

## Phase 4 - Lost Token Reissue

### Entry Point

Core route: `POST /api/tokens/reissue`

Request body:

```json
{
  "ballotId": "uuid",
  "voterIdentifier": "alice@example.com"
}
```

The endpoint is public and MUST be rate limited. Core currently applies
`strictRateLimiter`.

### When Reissue Is Allowed

A voter may request reissue only when all of these are true:

1. The ballot exists and is open.
2. `hashIdentifier(voterIdentifier)` matches an `EligibilityEntry` in the
   ballot's eligibility list.
3. `EligibilityEntry.tokenIssued = true`, or no prior token was issued and the
   request can safely fall back to normal issuance.
4. The system has not recorded a successful vote for the voter.
5. Rate limiting and abuse detection allow the request.

Because AnonVote deliberately does not link identifiers to token hashes, the
system must avoid claiming it can identify the exact lost token unless a future
implementation adds a privacy-preserving reissue handle. Current core searches
for an unused token in the ballot and replaces one unused token. A stricter
implementation MAY store an encrypted or blinded reissue handle, but it MUST
NOT create a direct join path from `EligibilityEntry` to `Vote`.

### Required Steps

1. Require `ballotId` and `voterIdentifier`.
2. Apply rate limiting before any database-intensive work.
3. Load the ballot and eligibility list.
4. Reject closed or unknown ballots using the same generic posture as token
   issuance.
5. Compute `identifierHash = hashIdentifier(voterIdentifier)`.
6. Look up `EligibilityEntry`.
7. If no entry exists, reject without revealing other voter state.
8. If `tokenIssued = false`, call the normal issuance flow.
9. Determine whether the previous token can be considered unused.
10. If already voted, reject with an `AlreadyVoted` style response.
11. Generate a replacement token:

    ```text
    newRawToken  = generateToken()
    newTokenHash = hashToken(newRawToken)
    ```

12. In a database transaction:
    - invalidate exactly one old unused token for the ballot
    - create the new `VoterToken { tokenHash: newTokenHash, ballotId }`
    - create an audit event for token issuance or reissue
13. Deliver `newRawToken` once, subject to the same delivery rules as Phase 2.

### Old Token Invalidation

Soft invalidation is preferred. The protocol-level representation is:

```text
old VoterToken.used = true
old VoterToken.usedAt = reissue timestamp
old VoterToken invalidation reason = "REISSUED" (if supported)
```

The current Prisma model does not include an invalidation reason, and current
core deletes one unused token during reissue. Future core implementations SHOULD
prefer soft invalidation so audits can distinguish a used vote token from a
revoked lost token without storing the token value.

### Rate Limiting

The reissue endpoint MUST be at least as strict as initial token issuance.
Recommended minimum controls:

- per-IP rate limit
- per-ballot rate limit
- per-normalized-identifier rate limit after hashing
- escalating delay or temporary lockout after repeated failed attempts

Rate-limit records MUST NOT store raw identifiers. If identifier-scoped rate
limiting is needed, store `hashIdentifier(voterIdentifier)`.

### Reissue Logging Rules

May log:

- `ballotId`
- audit event ID
- provider message ID for delivery
- whether reissue succeeded or failed
- generic failure category

Must not log:

- raw voter identifier
- raw token
- token hash
- encrypted vote payload
- plaintext option ID

---

## Phase 5 - Token Invalidation

Token invalidation is what enforces one token, one vote.

### Successful Vote Invalidation

After a valid vote is written, core MUST mark the effective `VoterToken` as:

```text
used = true
usedAt = current timestamp
```

This update MUST happen in the same transaction as the vote write and the
`VOTE_CAST` audit event.

### Why Soft Deletion Is Used

Successful vote tokens MUST be soft-invalidated rather than hard-deleted:

- A used token record proves that a duplicate vote attempt should be rejected.
- Audit counts can compare issued tokens, used tokens, and votes cast.
- Recount and incident response workflows can inspect state without needing raw
  token values.
- Hard deletion makes duplicate detection ambiguous.

Hard deletion is only acceptable for pre-vote token reissue if the system has a
separate audit record proving that the deleted token was revoked before use.

### Audit Logging

Token use is recorded as:

```text
AuditEvent { ballotId, eventType: "VOTE_CAST" }
```

Duplicate attempts are recorded as:

```text
AuditEvent { ballotId, eventType: "DUPLICATE_VOTE_ATTEMPT" }
AuditEvent { ballotId, eventType: "DUPLICATE_TOKEN_ATTEMPT" }
```

No audit event may contain the raw token, token hash, raw identifier, or
identifier hash.

---

## Sequence Diagram 1 - Happy Path

```text
Admin                Core API                 Database              Voter              Ledger
  |                     |                         |                    |                  |
  | POST /eligibility   |                         |                    |                  |
  | CSV identifiers     |                         |                    |                  |
  |-------------------->| hashIdentifier(id) x N  |                    |                  |
  |                     | create EligibilityList  |                    |                  |
  |                     | create EligibilityEntry |                    |                  |
  |                     |------------------------>|                    |                  |
  |<--------------------| eligibilityListId       |                    |                  |
  |                     |                         |                    |                  |
  | create ballot       |                         |                    |                  |
  |-------------------->| Ballot references list  |                    |                  |
  |                     |------------------------>|                    |                  |
  |                     |                         |                    |                  |
  |                     | POST /tokens            | voterIdentifier    |                  |
  |                     |<---------------------------------------------|                  |
  |                     | hashIdentifier(input)   |                    |                  |
  |                     | lookup EligibilityEntry |                    |                  |
  |                     |------------------------>|                    |                  |
  |                     | generateToken()         |                    |                  |
  |                     | hashToken(rawToken)     |                    |                  |
  |                     | tx: create VoterToken   |                    |                  |
  |                     | tx: tokenIssued = true  |                    |                  |
  |                     | tx: TOKEN_ISSUED event  |                    |                  |
  |                     |------------------------>|                    |                  |
  |                     | deliver rawToken once   |------------------->|                  |
  |                     | writeRecord TOKEN_ISSUED|--------------------------------------->|
  |                     |                         |                    |                  |
  |                     | POST /votes             | rawToken+optionId  |                  |
  |                     |<---------------------------------------------|                  |
  |                     | hashToken(rawToken)     |                    |                  |
  |                     | lookup VoterToken       |                    |                  |
  |                     | validate ballot/option  |                    |                  |
  |                     | encryptVote(optionId)   |                    |                  |
  |                     | tx: create Vote         |                    |                  |
  |                     | tx: mark token used     |                    |                  |
  |                     | tx: VOTE_CAST event     |                    |                  |
  |                     |------------------------>|                    |                  |
  |                     | vote confirmation       |------------------->|                  |
  |                     | writeRecord VOTE_CAST   |--------------------------------------->|
```

---

## Sequence Diagram 2 - Lost Token Reissue

```text
Voter                 Core API                 Database              Delivery
  |                     |                         |                    |
  | POST /tokens        |                         |                    |
  | voterIdentifier     |                         |                    |
  |-------------------->| hashIdentifier(input)   |                    |
  |                     | find EligibilityEntry   |                    |
  |                     | tokenIssued = true      |                    |
  |                     |------------------------>|                    |
  |<--------------------| TokenAlreadyIssued      |                    |
  |                     |                         |                    |
  | POST /tokens/reissue|                         |                    |
  | voterIdentifier     |                         |                    |
  |-------------------->| rate limit check        |                    |
  |                     | hashIdentifier(input)   |                    |
  |                     | verify eligibility      |                    |
  |                     | verify not already used |                    |
  |                     | generateToken()         |                    |
  |                     | hashToken(newRawToken)  |                    |
  |                     | tx: revoke old unused   |                    |
  |                     | tx: create new token    |                    |
  |                     | tx: audit reissue       |                    |
  |                     |------------------------>|                    |
  |                     | deliver newRawToken once|------------------->|
  |<--------------------| confirmation            |                    |
```

Reissue must not reveal whether any other voter has requested or used a token.

---

## Sequence Diagram 3 - Failed Redemption

```text
Voter                 Core API                 Database              Outcome
  |                     |                         |                    |
  | POST /votes         |                         |                    |
  | token + option      |                         |                    |
  |-------------------->| hashToken(token)        |                    |
  |                     | lookup VoterToken       |                    |
  |                     |------------------------>|                    |
  |                     |                         |                    |
  |                     | if token missing        |                    |
  |<--------------------| Invalid token           | no DB mutation      |
  |                     |                         |                    |
  |                     | if token used           |                    |
  |                     | create duplicate audit  |------------------->|
  |<--------------------| Token already used      | no Vote row         |
  |                     |                         |                    |
  |                     | if ballot closed/expired|                    |
  |<--------------------| Ballot closed           | token remains unused|
  |                     |                         |                    |
  |                     | if option invalid       |                    |
  |<--------------------| Invalid option          | token remains unused|
```

No failed redemption path may create a `Vote` row. Only duplicate-token attempts
may create a duplicate audit event, and that event must not contain the token.

---

## Cross-Repository Reference Map

| Concern | Reference |
| --- | --- |
| Identifier hashing | `AnonVote/js/src/crypto.ts` `hashIdentifier`; `AnonVote/core/backend/src/utils/crypto.ts` `hashIdentifier` |
| Token generation | `AnonVote/js/src/crypto.ts` `generateToken`; `AnonVote/core/backend/src/utils/crypto.ts` `generateToken` |
| Token hashing | `AnonVote/js/src/crypto.ts` `hashToken`; `AnonVote/core/backend/src/utils/crypto.ts` `hashToken` |
| Eligibility upload | `AnonVote/core/backend/src/routes/eligibility.ts` `POST /api/eligibility` |
| Token issuance | `AnonVote/core/backend/src/routes/tokens.ts` `POST /api/tokens`; `identityManager.issueToken` |
| Token reissue | `AnonVote/core/backend/src/routes/tokens.ts` `POST /api/tokens/reissue`; `identityManager.reissueToken` |
| Token reset | `AnonVote/core/backend/src/routes/tokens.ts` `POST /api/tokens/reset/:ballotId`; `identityManager.resetBallotTokens` |
| Vote submission | `AnonVote/core/backend/src/routes/votes.ts` `POST /api/votes`; `privacyEngine.submitVote` |
| Active ledger writes | `AnonVote/core/backend/src/services/stellarService.ts` `writeRecord` |
| Soroban-ready helpers | `AnonVote/core/backend/src/services/sorobanService.ts` `sorobanRecordToken`, `sorobanRecordVote` |
| Contract counters | `AnonVote/core/contracts/README.md` `record_token`, `record_vote`, `get_tokens_issued`, `get_votes_cast` |

---

## Implementation Checklist

- [ ] Eligibility ingestion stores only `identifierHash` values.
- [ ] Duplicate eligibility entries are detected after normalization and hashing.
- [ ] Token issuance calls `generateToken` before `hashToken` and stores only the
      hash.
- [ ] Raw token delivery is one-time and never logged.
- [ ] `POST /api/tokens` and `POST /api/tokens/reissue` are rate limited.
- [ ] Vote redemption validates by `hashToken(rawToken)`, never plaintext token
      comparison.
- [ ] Vote write, token invalidation, and `VOTE_CAST` audit creation are one
      database transaction.
- [ ] Failed redemption paths do not create `Vote` rows.
- [ ] Lost token reissue invalidates an old unused token before issuing a new
      token.
- [ ] Successful vote token invalidation is soft deletion (`used = true`,
      `usedAt` set).
- [ ] Audit events never store raw identifiers, identifier hashes, raw tokens,
      token hashes, plaintext options, or encrypted payloads.
