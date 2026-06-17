# Cryptographic Primitive Specifications

**Package:** [`@anonvote/crypto`](https://github.com/AnonVote/js)
**Source:** `src/crypto.ts`

---

## `hashIdentifier(id: string): string`

| Property            | Value                             |
| ------------------- | --------------------------------- |
| Algorithm           | SHA-256                           |
| Input preprocessing | `trim()` + `toLowerCase()`        |
| Output              | 64-character lowercase hex string |
| Deterministic       | Yes                               |
| Reversible          | No                                |

**Purpose:** Hash a voter identifier before storing in the eligibility list. The original is never persisted.

### Security property

**Preimage resistance.** An attacker who obtains the eligibility list cannot recover the original voter identifiers (email addresses, employee IDs, etc.). The normalization step (`trim()` + `toLowerCase()`) ensures that `"Alice@Example.com"` and `" alice@example.com "` produce the same hash, preventing duplicate eligibility entries for the same voter.

### Failure modes

| Failure | Consequence | Severity |
| ------- | ----------- | -------- |
| Using a non-collision-resistant hash (e.g., truncated SHA-256) | Two different identifiers could map to the same hash, allowing one voter to claim another's eligibility | Critical |
| Skipping normalization | `"alice@example.com"` and `"Alice@example.com"` produce different hashes, creating duplicate entries | Medium |
| Using a reversible encoding (e.g., Base64 without hashing) | Anyone with database access can read all voter identities | Critical |

### Implementation reference

```
// AnonVote/js — src/crypto.ts
export function hashIdentifier(id: string): string {
  return createHash('sha256').update(id.trim().toLowerCase()).digest('hex');
}
```

---

## `generateToken(): string`

| Property      | Value                             |
| ------------- | --------------------------------- |
| Source        | `crypto.randomBytes(32)` (CSPRNG) |
| Entropy       | 256 bits                          |
| Output        | 64-character lowercase hex string |
| Deterministic | No (fresh randomness each call)   |

**Purpose:** Generate a one-time anonymous voter token. The raw value is given to the voter and discarded. Only its hash is stored — see `hashToken`.

### Security property

**Unpredictability.** The token must be computationally indistinguishable from random to anyone who does not know the seed. This requires a cryptographically secure pseudo-random number generator (CSPRNG). `Math.random()`, `Date.now()`, or any predictable seed is unacceptable because an attacker who can guess tokens can impersonate voters.

### Failure modes

| Failure | Consequence | Severity |
| ------- | ----------- | -------- |
| Using a non-CSPRNG source (e.g., `Math.random()`) | Token values are predictable; attacker can enumerate valid tokens and vote as any voter | Critical |
| Using fewer than 32 bytes | Reduced entropy makes brute-force enumeration feasible | Critical |
| Reusing the same random bytes across calls | Two voters receive the same token, causing vote collisions | Critical |

### Implementation reference

```
// AnonVote/js — src/crypto.ts
import { randomBytes } from 'crypto';

export function generateToken(): string {
  return randomBytes(32).toString('hex');
}
```

---

## `hashToken(token: string): string`

| Property            | Value                              |
| ------------------- | ---------------------------------- |
| Algorithm           | SHA-256                            |
| Input preprocessing | None (raw token, no normalization) |
| Output              | 64-character lowercase hex string  |
| Deterministic       | Yes                                |
| Reversible          | No                                 |

**Purpose:** Hash the raw voter token for server-side storage. Note: unlike `hashIdentifier`, no normalization is applied — token hashing is case-sensitive and whitespace-sensitive by design.

### Security property

**Preimage resistance (same as `hashIdentifier`), plus second-preimage resistance.** An attacker who obtains the token hash database cannot recover the raw token, and therefore cannot vote with it. The two-step design (generate then hash) ensures that the raw token exists only transiently in memory and on the voter's device. The server never stores the raw token at any point.

### Why not normalize?

Token hashing deliberately omits normalization because tokens are exact byte strings, not human-readable identifiers. A voter who copies their token with an accidental trailing newline will produce a different hash, and that is correct behavior — the voter must copy the token exactly. Normalization would silently accept malformed tokens, which is a security risk.

### Failure modes

| Failure | Consequence | Severity |
| ------- | ----------- | -------- |
| Normalizing input (trim, lowercase) | An attacker who intercepts a near-match token (e.g., with a trailing space) could vote with a slightly different but accepted token | High |
| Storing the raw token on the server | Database compromise leaks all active tokens; attacker can cast votes before the legitimate voter | Critical |
| Using a weak hash algorithm | Token hash collisions could allow two tokens to map to the same stored hash | Critical |

### Implementation reference

```
// AnonVote/js — src/crypto.ts
export function hashToken(token: string): string {
  return createHash('sha256').update(token).digest('hex');
}
```

---

## `encryptVote(optionId: string, ballotKey: string): string`

| Property      | Value                                           |
| ------------- | ----------------------------------------------- |
| Algorithm     | AES-256-GCM                                     |
| Key           | 32 bytes (64-char hex string)                   |
| IV            | 12-byte CSPRNG (unique per call)                |
| Auth tag      | 16 bytes (GCM default)                          |
| Output format | `base64(iv):base64(authTag):base64(ciphertext)` |

**Purpose:** Encrypt a vote option ID before storing in the database. The encryption key is never stored in the database; it lives only in the environment config.

### Security property

**IND-CCA2 (Indistinguishability under adaptive chosen-ciphertext attack).** AES-256-GCM provides both confidentiality (the vote choice is hidden) and authenticity (tampering with the ciphertext is detectable). The 12-byte IV is generated fresh per call — never reuse an IV with the same key, as this breaks the security guarantee of GCM.

### Why GCM over CBC?

| Property | GCM | CBC |
| -------- | --- | --- |
| Authenticated encryption | Yes (built-in auth tag) | No (requires separate HMAC) |
| Parallelizable | Yes | No |
| IV length | 12 bytes (recommended) | 16 bytes |
| Tamper detection | Auth tag verified before decryption | Not provided |

GCM was chosen because it provides authenticated encryption in a single pass, eliminating the risk of missing a separate HMAC step. CBC with HMAC would also be secure but adds complexity and a larger attack surface (padding oracle attacks, HMAC comparison timing).

### Failure modes

| Failure | Consequence | Severity |
| ------- | ----------- | -------- |
| IV reuse with the same key | GCM loses all confidentiality; attacker can recover the plaintext | Critical |
| Stripping the auth tag before storage | Tampering with the ciphertext becomes undetectable | Critical |
| Using a weak key (e.g., derived from a password) | Brute-force key recovery becomes feasible | Critical |
| Skipping IV uniqueness enforcement | Two identical votes produce the same ciphertext (deterministic encryption), leaking voting patterns | High |

### Implementation reference

```
// AnonVote/js — src/crypto.ts
import { createCipheriv, randomBytes } from 'crypto';

export function encryptVote(optionId: string, ballotKey: string): string {
  const key = Buffer.from(ballotKey, 'hex');
  if (key.length !== 32) throw new Error('Invalid ballot key length');

  const iv = randomBytes(12);
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const encrypted = Buffer.concat([cipher.update(optionId, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();

  return `${iv.toString('base64')}:${authTag.toString('base64')}:${encrypted.toString('base64')}`;
}
```

---

## `decryptVote(payload: string, ballotKey: string): string`

| Property         | Value                                           |
| ---------------- | ----------------------------------------------- |
| Algorithm        | AES-256-GCM                                     |
| Input format     | `base64(iv):base64(authTag):base64(ciphertext)` |
| Tamper detection | GCM auth tag verified before decryption         |

**Purpose:** Decrypt a vote payload to recover the option ID during result tallying. Should only be called by the result engine.

### Security property

**Authenticated decryption.** GCM auth tag verification ensures that the ciphertext has not been tampered with since encryption. Failed verification means either:
- The ciphertext was corrupted in transit/storage, or
- An attacker attempted to modify the vote

In either case, the system must reject the result and not fall back to unauthenticated decryption.

### Why silent failure is unacceptable

If `decryptVote` silently returns a partial or incorrect result when auth tag verification fails, an attacker could:
1. Intercept a vote payload in transit
2. Modify the ciphertext bytes
3. Cause the tally to count a different option than the voter selected

The function **must** throw on auth tag failure so the calling code can handle the integrity breach explicitly.

### Failure modes

| Failure | Consequence | Severity |
| ------- | ----------- | -------- |
| Skipping auth tag verification | Tampered votes are silently decrypted to garbage values, corrupting the tally | Critical |
| Using a constant-time comparison for auth tag verification only | Authentication bypass not applicable here (GCM handles this internally) | Low |
| Returning partial plaintext on error | Caller may use incorrect data without realizing | Critical |

### Implementation reference

```
// AnonVote/js — src/crypto.ts
import { createDecipheriv } from 'crypto';

export function decryptVote(payload: string, ballotKey: string): string {
  const key = Buffer.from(ballotKey, 'hex');
  const [ivB64, authTagB64, encryptedB64] = payload.split(':');
  if (!ivB64 || !authTagB64 || !encryptedB64) throw new Error('Malformed payload');

  const iv = Buffer.from(ivB64, 'base64');
  const authTag = Buffer.from(authTagB64, 'base64');
  const encrypted = Buffer.from(encryptedB64, 'base64');

  const decipher = createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);
  return decipher.update(encrypted, undefined, 'utf8') + decipher.final('utf8');
}
```

---

## Dependency diagram

The five primitives compose into the full vote lifecycle as follows:

```
                  ┌─────────────────────────┐
                  │  Organization registers  │
                  │  (out of scope for       │
                  │   crypto layer)          │
                  └─────────┬───────────────┘
                            │
                            ▼
                  ┌─────────────────────────┐
                  │  Upload eligibility     │
                  │  list                   │
                  │                         │
                  │  ┌─────────────────┐    │
                  │  │ hashIdentifier  │◄───│── voter email/id
                  │  │ (SHA-256)       │    │
                  │  └────────┬────────┘    │
                  │           ▼             │
                  │   Stored as hash        │
                  └─────────┬───────────────┘
                            │
                            ▼
                  ┌─────────────────────────┐
                  │  Voter requests token   │
                  │                         │
                  │  ┌─────────────────┐    │
                  │  │ generateToken   │◄───│── CSPRNG
                  │  │ (256-bit)       │    │
                  │  └────────┬────────┘    │
                  │           ▼             │
                  │   Raw token → voter     │
                  │   (not stored)          │
                  │           │             │
                  │           ▼             │
                  │  ┌─────────────────┐    │
                  │  │  hashToken      │    │
                  │  │ (SHA-256)       │    │
                  │  └────────┬────────┘    │
                  │           ▼             │
                  │   Stored as hash        │
                  └─────────┬───────────────┘
                            │
                            ▼
                  ┌─────────────────────────┐
                  │  Voter casts vote       │
                  │                         │
                  │  ┌─────────────────┐    │
                  │  │  encryptVote    │◄───│── ballotKey + optionId
                  │  │ (AES-256-GCM)   │    │
                  │  └────────┬────────┘    │
                  │           ▼             │
                  │   Stored as:            │
                  │   iv:authTag:ciphertext │
                  └─────────┬───────────────┘
                            │
                            ▼
                  ┌─────────────────────────┐
                  │  Tally results          │
                  │                         │
                  │  ┌─────────────────┐    │
                  │  │  decryptVote    │◄───│── ballotKey
                  │  │ (AES-256-GCM)   │    │
                  │  └────────┬────────┘    │
                  │           ▼             │
                  │   Recovered optionId    │
                  └─────────────────────────┘
```

### Primitive dependency table

| Primitive         | Depends on              | Called by              |
| ----------------- | ----------------------- | ---------------------- |
| `hashIdentifier`  | SHA-256 (Node.js crypto) | Eligibility upload     |
| `generateToken`   | CSPRNG (Node.js crypto)  | Token issuance         |
| `hashToken`       | SHA-256 (Node.js crypto) | Token storage          |
| `encryptVote`     | AES-256-GCM + CSPRNG    | Vote submission        |
| `decryptVote`     | AES-256-GCM             | Result tallying        |

---

## Key management

`encryptVote` and `decryptVote` depend on a **per-ballot encryption key** (`ballotKey`). This section specifies how keys are generated, stored, and used.

### Key generation

- Each ballot receives a unique 256-bit (32-byte) encryption key at creation time.
- The key is generated using `crypto.randomBytes(32)` — the same CSPRNG used for token generation.
- The key is encoded as a 64-character lowercase hex string.

### Key storage

- The ballot key **must not** be stored in the application database alongside encrypted votes. Doing so would mean a database breach reveals both the ciphertext and the key, defeating encryption entirely.
- The ballot key is stored in a **separate, access-controlled secrets store** (e.g., environment config, vault, or cloud KMS).
- Access to the key should be restricted to:
  - The vote encryption service (write + read)
  - The tallying service (read only, and only during tally operations)

### Key lifecycle

| Stage | Action | Notes |
| ----- | ------ | ----- |
| Ballot creation | Generate key, store in secrets store | Key is never returned in the API response |
| Vote submission | Read key, encrypt, discard from memory | Key held in memory for the duration of a single request |
| Tallying | Read key, decrypt all votes, discard | Key may be cached for the duration of tally |
| Ballot deletion | Delete key from secrets store | Encrypted votes become permanently undecryptable |

### Key rotation

Ballot keys cannot be rotated transparently because existing encrypted votes would become unreadable. If rotation is required:
1. Generate a new key.
2. Decrypt all existing votes with the old key.
3. Re-encrypt all votes with the new key.
4. Delete the old key.

This is an expensive operation and should be avoided except in response to a confirmed key compromise.

---

## Threat model

This section documents what the AnonVote cryptographic layer **does** and **does not** protect against, so that integrators and auditors can make informed trust decisions.

### Protected against

| Threat | Mitigated by | Notes |
| ------ | ------------ | ----- |
| Database breach leaking vote choices | AES-256-GCM encryption of vote payloads | Attacker gets ciphertext only; no key in database |
| Database breach leaking voter identities | SHA-256 hashing of identifiers | Preimage resistance prevents recovery of raw identifiers |
| Database breach leaking active tokens | SHA-256 hashing of tokens | Token hash alone cannot be used to vote |
| Tampering with stored votes | GCM auth tag on encrypted votes | Tampered ciphertext rejected at decryption time |
| Token forgery | 256-bit CSPRNG tokens | 2^256 space makes enumeration infeasible |
| Duplicate eligibility | Normalized identifier hashing | Same voter always produces the same hash |

### NOT protected against

| Threat | Reason | Recommended mitigation |
| ------ | ------ | ---------------------- |
| **Traffic analysis** | The crypto layer does not pad or obfuscate message lengths or timing | Use Tor or a VPN for the deployment infrastructure |
| **Coercion** | A voter can be forced to reveal their token and prove how they voted | The system design cannot prevent this; consider using a coercion-resistant voting scheme for high-stakes elections |
| **Admin key compromise** | If the ballot key is stolen from the secrets store, all votes for that ballot can be decrypted | Restrict secrets store access; audit all key access events |
| **Side-channel attacks** | The implementation runs in Node.js, which does not provide constant-time guarantees for all operations | Not addressed by this spec; consider a hardware-backed HSM for production deployments |
| **Compromised client device** | If the voter's device is compromised, the raw token and vote choice are visible | Out of scope for the protocol layer; browser sandboxing and OS-level isolation |
| **Replay attacks on token issuance** | An attacker who intercepts a token request could request multiple tokens for the same voter | Rate-limit token requests; bind tokens to a specific ballot and voter identifier hash |
