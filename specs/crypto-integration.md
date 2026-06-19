# @anonvote/crypto Integration Guide

**Package:** [`@anonvote/crypto`](https://github.com/AnonVote/js)
**Reference implementation:** `AnonVote/js/src/crypto.ts`
**Primary consumers:** `AnonVote/core`, audit tooling, future SDKs

This guide defines how application code must compose the exported
`@anonvote/crypto` primitives. The primitive specification explains what each
function does in isolation; this document explains how to use them together
without weakening AnonVote's privacy model.

The package currently exports five primitives:

- `hashIdentifier(id: string): string`
- `generateToken(): string`
- `hashToken(token: string): string`
- `encryptVote(optionId: string, ballotKey: string): string`
- `decryptVote(payload: string, ballotKey: string): string`

## Integration Invariants

Every integration must preserve these invariants:

1. Raw voter identifiers are accepted only at trust boundaries and must not be
   persisted or logged.
2. Raw voter tokens are returned to the voter once and must not be persisted or
   logged.
3. Token validation is always `hashToken(rawToken)` followed by a stored-hash
   comparison. Never compare raw tokens server-side.
4. Vote encryption uses a per-ballot AES-256 key. A global application-wide vote
   encryption key is not compatible with this guide.
5. Failed `decryptVote` authentication is a critical tally failure. Do not catch
   the error and continue producing a partial tally.
6. Database rows for submitted votes must not store plaintext vote choices as
   voting facts beside the encrypted payload.
7. `@anonvote/crypto` should be imported directly by consumers instead of
   copy-pasting local versions of the same functions.

## Identity-to-Hash Flow

Use this flow when importing an eligibility list or when a voter submits an
identifier to request a token.

### Sequence

1. Receive a raw identifier at the API boundary.
2. Reject empty input before hashing.
3. Call `hashIdentifier(rawIdentifier)`.
4. Store only the resulting `identifierHash`.
5. Drop the raw identifier from the request context as soon as the response is
   formed.
6. Do not log the raw identifier. Do not include it in audit events.

### Minimal Tested Example

```ts
import { hashIdentifier } from "@anonvote/crypto";

const uploaded = "  Alice@Example.COM  ";
const identifierHash = hashIdentifier(uploaded);

if (identifierHash !== hashIdentifier("alice@example.com")) {
  throw new Error("identifier normalization drift");
}

// Store identifierHash in EligibilityEntry.identifierHash.
// Do not store uploaded or any other raw identifier value.
```

### Storage Contract

| Value | Store? | Notes |
| --- | --- | --- |
| Raw identifier | No | May appear only in request memory. |
| Normalized identifier | No | Internal to `hashIdentifier`. |
| `identifierHash` | Yes | Store in `EligibilityEntry.identifierHash`. |
| Identifier hash in audit event | Avoid | Prefer aggregate audit events without per-voter values. |

### Normalization Awareness

`hashIdentifier` applies `trim().toLowerCase()` before SHA-256. Callers should
not pre-normalize differently. For example, do not apply locale-specific
case-folding in one service and ASCII lowercasing in another; the package
function is the canonical normalization point.

## Token Lifecycle Flow

Use this flow for token issuance, delivery, redemption, and invalidation.

### Issuance Sequence

1. Validate that `identifierHash` is present in the ballot eligibility list.
2. Verify that no active token has already been issued for that eligibility
   entry.
3. Call `generateToken()` to create a raw 32-byte token encoded as 64 lowercase
   hex characters.
4. Immediately call `hashToken(rawToken)`.
5. Store only the token hash.
6. Mark the eligibility entry as token-issued in the same database transaction.
7. Deliver the raw token to the voter exactly once.
8. Do not log, store, or audit the raw token.

### Redemption Sequence

1. Receive `rawToken` and `ballotId` from the vote submission request.
2. Call `hashToken(rawToken)`.
3. Look up `VoterToken.tokenHash`.
4. Validate that the token belongs to the ballot and is unused.
5. Accept the vote inside a transaction that also marks the token as used.
6. Discard the raw token before writing audit records or returning the response.

### Minimal Tested Example

```ts
import { generateToken, hashToken } from "@anonvote/crypto";

const rawToken = generateToken();
const tokenHash = hashToken(rawToken);

await tx.voterToken.create({
  data: {
    ballotId,
    tokenHash,
    used: false,
  },
});

const submittedHash = hashToken(rawToken);
if (submittedHash !== tokenHash) {
  throw new Error("token hash mismatch");
}

// Store tokenHash in VoterToken.tokenHash.
// Return rawToken to the voter once, then discard it.
```

### Storage Contract

| Value | Store? | Notes |
| --- | --- | --- |
| Raw token | No | Return once to the voter. |
| `hashToken(rawToken)` | Yes | Store as `VoterToken.tokenHash`. |
| Token delivery metadata | Yes | May store aggregate event type and timestamp, but not raw token. |
| Used flag | Yes | Set atomically with accepted vote. |

### Atomicity Requirement

Vote acceptance must be a single transaction covering:

- token lookup by hash
- token unused check
- encrypted vote write
- token invalidation
- audit event creation

If any step fails, the token must remain unused and the vote must not be
committed.

## Vote Encryption Flow

Use this flow when accepting a vote and later tallying the ballot.

### Submission Sequence

1. Validate the raw token with the token lifecycle flow.
2. Validate that `optionId` belongs to `ballotId`.
3. Load the per-ballot encryption key from secure storage.
4. Call `encryptVote(optionId, ballotKey)`.
5. Store the returned encrypted payload.
6. Do not store the plaintext `optionId` as the persisted voting fact.
7. Mark the token as used in the same transaction.

### Tally Sequence

1. Load the per-ballot encryption key from secure storage.
2. For each stored encrypted vote payload, call `decryptVote(payload, ballotKey)`.
3. If decryption throws, abort the tally and mark the ballot for investigation.
4. Validate the decrypted option ID against the ballot's option set.
5. Aggregate the decrypted option IDs.
6. Store the final tally and result hash.

### Minimal Tested Example

```ts
import { randomBytes } from "crypto";
import { encryptVote, decryptVote } from "@anonvote/crypto";

const ballotKey = randomBytes(32).toString("hex");
const encryptedPayload = encryptVote("option-uuid-123", ballotKey);
const decryptedOptionId = decryptVote(encryptedPayload, ballotKey);

if (decryptedOptionId !== "option-uuid-123") {
  throw new Error("vote encryption round trip failed");
}
```

### Storage Contract

| Value | Store? | Notes |
| --- | --- | --- |
| `encryptedPayload` | Yes | Format is `base64(iv):base64(authTag):base64(ciphertext)`. |
| Plaintext `optionId` as vote fact | No | Keep only references needed for ballot definition, not submitted choice. |
| IV | Yes, inside payload | Generated by `encryptVote`; callers do not supply it. |
| Auth tag | Yes, inside payload | Required for tamper detection during tally. |
| Ballot encryption key | No database storage | Store in an environment-scoped secret manager. |

## Per-Ballot Key Management

Each ballot must have its own AES-256 encryption key. The key protects all vote
payloads for that ballot and must not be shared across ballots.

### Key Generation

Generate one key when the ballot is created:

```ts
import { randomBytes } from "crypto";

const ballotKey = randomBytes(32).toString("hex");
```

The output is a 64-character hex string representing 32 random bytes.

### Key Storage

Store the key in environment-scoped secure storage, such as a cloud secret
manager, sealed deployment secret, or hardware-backed vault.

Do not store the key:

- in the application database beside votes
- in the protocol repository
- in GitHub Actions logs
- in API responses
- in audit events
- in Stellar or Soroban records

The database may store a non-secret key reference such as
`ballotEncryptionKeyRef`. The reference identifies where to load the secret; it
must not contain the secret.

### Key Scope

| Scope | Allowed? | Reason |
| --- | --- | --- |
| One key per ballot | Yes | Limits compromise to one ballot. |
| One key per organization | No | Compromise reveals multiple ballots. |
| One global application key | No | Compromise reveals every encrypted vote ever stored with that key. |
| One key per vote | Not required | Adds operational complexity without matching current package API. |

### Rotation

Before voting opens:

- Generate a new per-ballot key.
- Replace the secret storage entry.
- No vote payload migration is needed because no votes exist.

After encrypted votes exist:

- Rotation requires decrypting every existing payload with the old key,
  encrypting with the new key, and committing the migrated payloads in an
  auditable maintenance window.
- If the old key is suspected compromised, treat the ballot privacy guarantee as
  compromised for that ballot. Rotation prevents future exposure but cannot
  erase access that already happened.

After result publication:

- Rotation does not restore ballot privacy if the key was compromised before
  or during tally.
- Retain the key only as long as policy requires independent recounts.

## Common Misuse Patterns

These are based on current AnonVote/core integration risks, not generic
examples.

### 1. Copying crypto functions instead of importing the package

`AnonVote/core/backend/src/utils/crypto.ts` contains a local copy of the crypto
functions. This creates drift from `@anonvote/crypto`; for example, the package
implementation checks `encryptVote` key length, while the local copy currently
passes the key directly into `Buffer.from`.

Correct integration:

```ts
import {
  hashIdentifier,
  generateToken,
  hashToken,
  encryptVote,
  decryptVote,
} from "@anonvote/crypto";
```

Why it matters:

- validation behavior must be identical across services
- package security fixes must reach consumers immediately
- tests against the package must reflect production behavior

### 2. Logging raw voter identifiers during token issuance

`AnonVote/core/backend/src/services/identityManager.ts` currently logs the raw
`voterIdentifier` during token issuance debugging. This breaks the privacy
boundary even though `hashIdentifier` is used for database storage.

Incorrect pattern:

```ts
console.log("voterIdentifier:", voterIdentifier);
```

Correct pattern:

```ts
console.log("token request received", { ballotId });
```

Why it matters:

- logs often have broader access than the database
- log retention can outlive ballot retention
- raw identifiers can be linked to token request timing

### 3. Using one global vote encryption key

`AnonVote/core/backend/src/config.ts` currently exposes one
`BALLOT_ENCRYPTION_KEY` value. That is operationally simple but does not satisfy
the per-ballot key requirement in this guide.

Incorrect pattern:

```ts
const encryptedPayload = encryptVote(optionId, config.ballotEncryptionKey);
```

Correct pattern:

```ts
const ballotKey = await keyStore.loadBallotKey(ballotId);
const encryptedPayload = encryptVote(optionId, ballotKey);
```

Why it matters:

- a global key compromise exposes every ballot encrypted with that key
- per-ballot keys limit blast radius
- per-ballot key references make rotation and incident response possible

### 4. Storing plaintext vote choices beside encrypted payloads

`AnonVote/core/backend/prisma/schema.prisma` currently stores `Vote.optionId`
beside `Vote.encryptedPayload`. If `optionId` represents the submitted choice,
then encryption no longer protects vote content at rest.

Incorrect pattern:

```ts
await tx.vote.create({
  data: { ballotId, optionId, encryptedPayload },
});
```

Correct pattern:

```ts
await tx.vote.create({
  data: { ballotId, encryptedPayload },
});
```

Why it matters:

- AES-GCM cannot protect data that is also stored in plaintext
- database readers can tally the ballot without the key
- public privacy claims become dependent on application convention instead of
  storage design

If the database needs relational validation, perform it before encryption and
store a non-choice structural reference only when it cannot reveal the vote.

### 5. Catching `decryptVote` auth failures and continuing the tally

`AnonVote/core/backend/src/services/resultEngine.ts` currently catches
decryption errors per vote and continues tallying. AES-GCM authentication
failure means the payload was malformed, tampered with, or decrypted with the
wrong key. Producing a partial tally after that failure is unsafe.

Incorrect pattern:

```ts
try {
  tally[decryptVote(vote.encryptedPayload, ballotKey)] += vote.weight;
} catch (err) {
  console.error(err);
}
```

Correct pattern:

```ts
const optionId = decryptVote(vote.encryptedPayload, ballotKey);
if (!(optionId in tally)) {
  throw new Error("decrypted option does not belong to ballot");
}
tally[optionId] += vote.weight;
```

Why it matters:

- auth tag failure is an integrity failure, not a recoverable missing vote
- continuing hides data corruption from administrators and voters
- a partial tally can be published as if it were complete

### 6. Reusing or externally supplying IVs

The current `encryptVote` API generates a fresh 12-byte IV internally. Callers
must not fork or wrap the function to pass a caller-controlled IV.

Incorrect pattern:

```ts
// Do not add this API.
encryptVoteWithIv(optionId, ballotKey, reusedIv);
```

Correct pattern:

```ts
const encryptedPayload = encryptVote(optionId, ballotKey);
```

Why it matters:

- AES-GCM requires a unique IV for each encryption under the same key
- IV reuse can reveal relationships between plaintexts and weaken integrity
- callers do not need to manage IVs because the package already does it

## Implementation Checklist

Before merging an integration of `@anonvote/crypto`, verify:

- consumer code imports from `@anonvote/crypto`
- no raw identifier logging remains
- no raw token logging or persistence exists
- token issuance stores only `hashToken(rawToken)`
- token redemption compares by recomputing `hashToken(submittedRawToken)`
- vote submission stores `encryptedPayload` without plaintext vote content
- each ballot has its own encryption key or key reference
- decryption errors abort tally publication
- test fixtures include tampered ciphertext and wrong-key failures
- examples in this guide still pass against the package implementation

## Reference Map

| Concern | Reference |
| --- | --- |
| Package primitives | `AnonVote/js/src/crypto.ts` |
| Package tests | `AnonVote/js/tests/crypto.test.ts` |
| Core identity flow | `AnonVote/core/backend/src/services/identityManager.ts` |
| Core vote submission flow | `AnonVote/core/backend/src/services/privacyEngine.ts` |
| Core tally flow | `AnonVote/core/backend/src/services/resultEngine.ts` |
| Core database schema | `AnonVote/core/backend/prisma/schema.prisma` |
| Core environment config | `AnonVote/core/backend/src/config.ts` |
