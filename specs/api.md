# REST API Specification

**Base URL:** `https://your-deployment/api`  
**Auth:** JWT via HTTP-only cookie (`session` header set on login)

---

## Organizations

| Method  | Endpoint                  | Auth    | Description                      |
| ------- | ------------------------- | ------- | -------------------------------- |
| `POST`  | `/organizations`          | —       | Register a new organization      |
| `POST`  | `/organizations/login`    | —       | Admin login; sets session cookie |
| `POST`  | `/organizations/logout`   | Session | Clears session cookie            |
| `GET`   | `/organizations/me`       | Session | Get current org profile          |
| `PATCH` | `/organizations/me`       | Session | Update org name or email         |
| `PATCH` | `/organizations/password` | Session | Change password                  |

---

## Ballots

| Method   | Endpoint       | Auth    | Description                                               |
| -------- | -------------- | ------- | --------------------------------------------------------- |
| `GET`    | `/ballots`     | Session | List all ballots for the authenticated org                |
| `POST`   | `/ballots`     | Session | Create a new ballot                                       |
| `GET`    | `/ballots/:id` | —       | Get a ballot by ID (public)                               |
| `PATCH`  | `/ballots/:id` | Session | Edit ballot topic, deadline, options, or eligibility list |
| `DELETE` | `/ballots/:id` | Session | Delete a ballot and all associated data                   |

### Create ballot request body

```json
{
  "topic": "Should we adopt remote-first?",
  "options": ["Yes", "No", "Abstain"],
  "deadline": "2026-07-01T00:00:00.000Z",
  "eligibilityListId": "uuid",
  "allowWeightedVoting": false,
  "allowRankedChoice": false,
  "maxRankings": null
}
```

---

## Eligibility

| Method | Endpoint       | Auth    | Description                                     |
| ------ | -------------- | ------- | ----------------------------------------------- |
| `POST` | `/eligibility` | Session | Upload voter list (multipart CSV or plain text) |

Identifiers are SHA-256 hashed server-side. Raw identifiers are never stored.

---

## Tokens

| Method | Endpoint                  | Auth    | Description                                         |
| ------ | ------------------------- | ------- | --------------------------------------------------- |
| `POST` | `/tokens`                 | —       | Request a one-time voter token                      |
| `POST` | `/tokens/reissue`         | —       | Reissue a lost token (blocked if vote already cast) |
| `POST` | `/tokens/reset/:ballotId` | Session | Reset all tokenIssued flags for a ballot (admin)    |

### Token request body

```json
{
  "ballotId": "uuid",
  "voterIdentifier": "alice@example.com"
}
```

### Token response

```json
{
  "data": {
    "token": "64-char-hex-string",
    "weight": 1
  }
}
```

---

## Votes

| Method | Endpoint | Auth | Description              |
| ------ | -------- | ---- | ------------------------ |
| `POST` | `/votes` | —    | Submit an anonymous vote |

### Vote request body

```json
{
  "ballotId": "uuid",
  "voterToken": "64-char-hex-string",
  "optionId": "uuid",
  "weight": 1,
  "rank": null
}
```

---

## Results

| Method | Endpoint                   | Auth    | Description                       |
| ------ | -------------------------- | ------- | --------------------------------- |
| `GET`  | `/results/:ballotId`       | —       | Get published result (public)     |
| `POST` | `/results/:ballotId/tally` | Session | Manually close and tally a ballot |

---

## Audit

| Method | Endpoint           | Auth | Description                                        |
| ------ | ------------------ | ---- | -------------------------------------------------- |
| `GET`  | `/audit/:ballotId` | —    | Get audit event counts and Stellar transaction IDs |

---

## Delegations

| Method | Endpoint       | Auth | Description                                   |
| ------ | -------------- | ---- | --------------------------------------------- |
| `POST` | `/delegations` | —    | Delegate voting power to another token holder |

---

## Verification

| Method | Endpoint                 | Auth | Description                             |
| ------ | ------------------------ | ---- | --------------------------------------- |
| `POST` | `/verification/generate` | —    | Generate a verification hash for a vote |
| `POST` | `/verification/verify`   | —    | Verify a vote using its hash            |

---

## Admin

| Method  | Endpoint               | Auth    | Description                            |
| ------- | ---------------------- | ------- | -------------------------------------- |
| `GET`   | `/admin/rate-limit`    | Session | Get current rate limit settings        |
| `PATCH` | `/admin/rate-limit`    | Session | Update rate limit preset               |
| `GET`   | `/admin/tokens-issued` | Session | Total tokens issued across all ballots |

---

## Error format

All errors follow a consistent envelope:

```json
{
  "error": "BadRequest",
  "message": "Human-readable description"
}
```

| Status | Error key            | When                                     |
| ------ | -------------------- | ---------------------------------------- |
| 400    | `BadRequest`         | Invalid input                            |
| 401    | `Unauthorized`       | Missing or expired session               |
| 403    | `Forbidden`          | Authenticated but not permitted          |
| 404    | `NotFound`           | Resource not found                       |
| 409    | `AlreadyVoted`       | Token already used to vote               |
| 409    | `TokenAlreadyIssued` | Token already issued for this identifier |
