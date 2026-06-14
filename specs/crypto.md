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

---

## `generateToken(): string`

| Property      | Value                             |
| ------------- | --------------------------------- |
| Source        | `crypto.randomBytes(32)` (CSPRNG) |
| Entropy       | 256 bits                          |
| Output        | 64-character lowercase hex string |
| Deterministic | No (fresh randomness each call)   |

**Purpose:** Generate a one-time anonymous voter token. The raw value is given to the voter and discarded. Only its hash is stored — see `hashToken`.

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

**Throws** if `ballotKey` is not exactly 64 hex characters.

---

## `decryptVote(payload: string, ballotKey: string): string`

| Property         | Value                                           |
| ---------------- | ----------------------------------------------- |
| Algorithm        | AES-256-GCM                                     |
| Input format     | `base64(iv):base64(authTag):base64(ciphertext)` |
| Tamper detection | GCM auth tag verified before decryption         |

**Purpose:** Decrypt a vote payload to recover the option ID during result tallying. Should only be called by the result engine.

**Throws** if the payload is malformed, the auth tag verification fails (tampered ciphertext), or the key is incorrect.
