# Importers API

Reference documentation for the importer endpoints in the TariffShield API.

Base path: `/importers`

All endpoints require `Authorization: Bearer <token>` and the caller must own the importer (unless noted).

---

## Collateral Status

### `GET /importers/:id/collateral-status`

Returns the collateral health of an importer's on-chain account, including whether the account is at risk of falling below the required threshold.

#### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `stale` | `boolean` | `true` if the on-chain collateral record has not been updated within 365 days (considered stale) |
| `lastUpdated` | `string` (ISO 8601) | Timestamp of the last collateral update on-chain |
| `expiresAt` | `string` (ISO 8601) | Timestamp when the current collateral record expires (`lastUpdated + 365 days`) |

#### Example — below-threshold (stale) response

```json
{
  "stale": true,
  "lastUpdated": "2024-01-15T08:30:00.000Z",
  "expiresAt": "2025-01-14T08:30:00.000Z"
}
```

> **Tip:** If `stale` is `true`, the importer should submit a new tariff CSV upload or deposit additional collateral to refresh the on-chain record. See the [SDK tutorial](../sdk-tutorial.md) for the collateral top-up walkthrough.

---

## Transaction Status Polling

### `GET /importers/:id/tx-status/:jobId`

Async blockchain operations (deposits, auto-top-ups, withdrawals, tariff uploads) return a `jobId` immediately. Poll this endpoint to track the transaction through to completion.

#### Job status values

| State | Meaning | HTTP Status |
|-------|---------|-------------|
| `waiting` | Queued, not yet picked up by a worker | 200 |
| `active` | Worker is processing the transaction | 200 |
| `completed` | Transaction submitted successfully on-chain | 200 |
| `failed` | Transaction failed (contract error, timeout, insufficient balance) | 400 |

#### Recommended polling guidance

| Parameter | Recommendation |
|-----------|---------------|
| Polling interval | 2–5 seconds |
| Timeout | 2 minutes (Stellar network typically finalizes in 3–5 seconds) |
| Max poll attempts | ~24 at 5-second intervals |

After `completed`, the response body contains the transaction hash and result. After `failed`, the response includes an `error` field with the failure reason.

#### Example request/response sequence

**1. Initiate a deposit (returns immediately):**

```
POST /importers/550e8400-e29b-41d4-a716-446655440000/deposit
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json

{ "amountStroops": "500000000", "bucket": "collateral" }
```

```json
{
  "jobId": "job_abc123",
  "statusUrl": "/importers/550e8400-e29b-41d4-a716-446655440000/tx-status/job_abc123"
}
```

**2. Poll the status URL:**

```
GET /importers/550e8400-e29b-41d4-a716-446655440000/tx-status/job_abc123
Authorization: Bearer eyJhbGciOi...
```

**In progress:**

```json
{ "state": "active", "progress": 50 }
```

**Completed:**

```json
{
  "state": "completed",
  "result": {
    "txHash": "a1b2c3d4e5f6...",
    "txUrl": "https://stellar.expert/explorer/testnet/tx/a1b2c3d4e5f6..."
  }
}
```

**Failed:**

```json
{
  "state": "failed",
  "error": "Transaction failed: insufficient balance"
}
```
