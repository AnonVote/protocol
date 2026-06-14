# Smart Contract Specification

**Repo:** [`AnonVote/contracts`](https://github.com/AnonVote/contracts)  
**Runtime:** Soroban (Stellar)  
**Language:** Rust (soroban-sdk 21.0.0)

---

## Purpose

The AnonVote Soroban contract provides on-chain queryable state that complements the off-chain privacy engine. It stores only counts and hashes — never voter identities, token values, or vote content.

---

## Contract: `AnonVoteContract`

### Storage keys

| Key                  | Type      | Description                                  |
| -------------------- | --------- | -------------------------------------------- |
| `Admin`              | `Address` | Authorized admin; only admin may write state |
| `BallotExists(hash)` | `bool`    | Tracks whether a ballot has been registered  |
| `TokensIssued(hash)` | `u32`     | Running count of tokens issued per ballot    |
| `VotesCast(hash)`    | `u32`     | Running count of votes cast per ballot       |
| `ResultHash(hash)`   | `String`  | SHA-256 of tally JSON; immutable once set    |

All keys use `ballot_id_hash` = SHA-256 hex of the ballot UUID. Raw IDs are never stored on-chain.

---

### Write functions (admin only)

#### `initialize(admin: Address)`

One-time initialization. Sets the admin address. Panics if already initialized.

#### `record_ballot(caller: Address, ballot_id_hash: String)`

Register a ballot on-chain. Initializes `TokensIssued` and `VotesCast` to 0. Panics if ballot already recorded.

#### `record_token(caller: Address, ballot_id_hash: String)`

Increment `TokensIssued` for a ballot. Panics if ballot not found.

#### `record_vote(caller: Address, ballot_id_hash: String)`

Increment `VotesCast` for a ballot. Panics if ballot not found.

#### `record_result(caller: Address, ballot_id_hash: String, result_hash: String)`

Record the SHA-256 of the tally JSON. Immutable once set — panics if result already recorded.

---

### Read functions (public, view calls)

#### `get_tokens_issued(ballot_id_hash: String) → u32`

Returns the number of tokens issued. View call — no transaction required.

#### `get_votes_cast(ballot_id_hash: String) → u32`

Returns the number of votes cast. View call.

#### `get_result_hash(ballot_id_hash: String) → Option<String>`

Returns the result hash, or `None` if results haven't been published yet.

#### `ballot_exists(ballot_id_hash: String) → bool`

Returns whether a ballot has been registered on-chain.

#### `is_consistent(ballot_id_hash: String) → bool`

Returns `true` if `tokens_issued == votes_cast`. View call.

---

## Integration with core

The TypeScript service in `AnonVote/contracts/service/sorobanService.ts` wraps these contract methods. Once `SOROBAN_CONTRACT_ID` is set in the backend `.env`, wire the helpers in:

| Backend service                | Contract call                                           |
| ------------------------------ | ------------------------------------------------------- |
| `ballotEngine.createBallot()`  | `sorobanRecordBallot(config, ballotIdHash)`             |
| `identityManager.issueToken()` | `sorobanRecordToken(config, ballotIdHash)`              |
| `privacyEngine.submitVote()`   | `sorobanRecordVote(config, ballotIdHash)`               |
| `resultEngine.tallyBallot()`   | `sorobanRecordResult(config, ballotIdHash, resultHash)` |

The `ballotIdHash` argument is `hashIdentifier(ballotId)` from `@anonvote/crypto`.

---

## Privacy guarantees

The contract enforces the same privacy model as the off-chain system:

- Only counts and hashes are stored — no identities, no token values, no vote content
- `record_result` is immutable — once a result hash is set, it cannot be changed
- Read functions are public — anyone can verify counts without authenticating
