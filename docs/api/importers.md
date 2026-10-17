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

**Notes**:
- Events are immutable and retained permanently for regulatory audit
- Sensitive data (e.g., API keys, passwords) is never included in event metadata
- Events are returned in reverse chronological order (newest first)

---

## Tariff CSV Upload

### POST /importers/:id/upload-tariff-csv

Upload tariff data for an importer. The upload triggers collateral requirement computation and updates the on-chain required collateral value.

**Authorization**: Importer owner or surety admin

**KYC Requirement**: Importer must have `kyc_status` of `approved`

**Request Body** (JSON):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filename` | string | No | Original filename for reference |
| `lineItems` | array | Yes | Array of tariff line items |

**Line Item Object**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sku` | string | No* | Product SKU (resolved against importer's SKU mappings) |
| `htsCode` | string | No* | HTS (Harmonized Tariff Schedule) code |
| `value` | number | Yes | Item value (must be positive) |
| `dutyRate` | number | No | Duty rate as decimal (e.g., 0.05 for 5%) |

*Either `sku` or `htsCode` must be provided. If `sku` is provided, it is resolved against the importer's active SKU-to-HTS mappings.

**Validation Behavior**:
1. **SKU Resolution**: SKUs are resolved against the importer's active catalog mappings. Unmapped SKUs result in a `422` error.
2. **HTS Rate Validation**: Declared duty rates are cross-referenced against the USITC HTS schedule. Underreported rates are flagged.
3. **CBP Validation**: Duty rates are validated against CBP rates. Deviations >10% are flagged (blocking or warning based on `CBP_VALIDATION_MODE`).
4. **Collateral Computation**: `bondFaceValue = annualDutyTotal × 0.1`, `requiredCollateral = bondFaceValue × 0.5`

**Example Request**:
```bash
curl -X POST https://api.tariffshield.com/api/v1/importers/abc123/upload-tariff-csv \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "q1-2024-tariffs.csv",
    "lineItems": [
      { "sku": "WIDGET-001", "value": 50000, "dutyRate": 0.05 },
      { "htsCode": "8471.30.01", "value": 120000, "dutyRate": 0.0 },
      { "sku": "GADGET-002", "value": 75000, "dutyRate": 0.0325 }
    ]
  }'
```

**Response** (200 OK):
```json
{
  "annualDutyTotal": 7625.00,
  "bondFaceValue": 762.50,
  "requiredCollateralStroops": "3812500000",
  "txHash": "abc123...def456",
  "txUrl": "https://stellar.expert/explorer/testnet/tx/abc123...def456",
  "htsWarnings": [
    {
      "htsCode": "8471.30.01",
      "status": "warning",
      "declaredRate": 0.0,
      "statutoryRate": 0.0,
      "message": "Rate matches statutory rate"
    }
  ]
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input (malformed line items)
- `403 Forbidden`: KYC not approved — includes `kycStatus` field
- `404 Not Found`: Importer not found
- `422 Unprocessable Entity`:
  - Unmapped SKUs found (includes `unmappedSkus` array)
  - HTS rate validation failed (includes `flagged` array with details)
  - CBP validation failed (includes `report` array with deviations)
- `429 Too Many Requests`: Rate limit exceeded — collateral requirements can only be updated once per 24 hours

**Notes**:
- The on-chain collateral update is submitted asynchronously
- Tariff spike alerts are evaluated after each successful upload
- The importer metrics view is refreshed after each upload

---

## Collateral Withdrawal

### POST /importers/:id/withdraw

Withdraw collateral funds from an importer's Stellar account. Supports immediate withdrawals, scheduled future withdrawals, and dual sign-off approval flows.

**Authorization**: Importer owner or surety admin

**KYC Requirement**: Importer must have `kyc_status` of `approved`

**AML Screening**: The importer's Stellar address is screened before processing. High-risk addresses are blocked.

**Request Body** (JSON):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amountStroops` | string\|number | Yes | Amount to withdraw in stroops (1 XLM = 10,000,000 stroops). Must be a positive integer. |
| `targetDate` | string | No | ISO 8601 future timestamp for scheduled withdrawal |
| `targetAddress` | string | No | Target Stellar address (defaults to importer's own address) |

**Withdrawal Flows**:

1. **Immediate Withdrawal** (default): If no `targetDate` is provided and the amount is below the dual sign-off threshold, the withdrawal is processed immediately via an async job.

2. **Scheduled Withdrawal**: If `targetDate` is provided, a future-dated scheduled withdrawal is created. The withdrawal will be executed on or after the target date.

3. **Dual Sign-Off Approval**: If dual approval is enabled and the amount meets or exceeds the configured threshold, a withdrawal request is created and requires a second approver's sign-off before execution.

**Example Request** (immediate):
```bash
curl -X POST https://api.tariffshield.com/api/v1/importers/abc123/withdraw \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "amountStroops": "500000000",
    "targetAddress": "GABC...XYZ"
  }'
```

**Response** (202 Accepted — immediate):
```json
{
  "jobId": "job_xyz789",
  "statusUrl": "/importers/abc123/tx-status/job_xyz789"
}
```

**Response** (201 Created — scheduled):
```json
{
  "status": "scheduled",
  "scheduledWithdrawal": {
    "id": "sw_123",
    "importerId": "abc123",
    "requestedBy": "user_123",
    "amountStroops": "500000000",
    "targetDate": "2024-02-01T00:00:00Z",
    "targetAddress": "GABC...XYZ",
    "status": "pending",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

**Response** (202 Accepted — pending approval):
```json
{
  "status": "pending_approval",
  "withdrawalRequestId": "wr_456",
  "amountStroops": "500000000",
  "message": "Withdrawal amount exceeds dual sign-off threshold and requires second approval."
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input (non-positive amount, invalid targetDate)
- `403 Forbidden`: KYC not approved or AML review required
- `404 Not Found`: Importer not found

**Notes**:
- The `targetDate` must be a valid future timestamp
- Dual sign-off threshold is configured per-importer
- Use the `statusUrl` to poll for transaction completion

---

## Bonds List

### GET /importers/:id/bonds

Retrieve the full bond history for an importer. Returns all bonds sorted by creation date (newest first).

**Authorization**: Importer owner or surety admin

**Parameters**:
- `id` (path, required): Importer ID

**Query Parameters**: None — this endpoint returns all bonds without pagination.

**Example Request**:
```bash
curl -X GET https://api.tariffshield.com/api/v1/importers/abc123/bonds \
  -H "Authorization: Bearer <token>"
```

**Response** (200 OK):
```json
{
  "bonds": [
    {
      "id": "bond_uuid_1",
      "bondNumber": "CB-2024-001",
      "policyType": "single_entry",
      "coverageAmount": "50000.00",
      "status": "active",
      "issuedAt": "2024-01-15T00:00:00Z",
      "expiresAt": "2025-01-15T00:00:00Z",
      "replacedById": null,
      "stellarContractAddress": "CABC...XYZ",
      "createdAt": "2024-01-15T10:30:00Z"
    },
    {
      "id": "bond_uuid_2",
      "bondNumber": "CB-2023-045",
      "policyType": "continuous",
      "coverageAmount": "100000.00",
      "status": "expired",
      "issuedAt": "2023-01-10T00:00:00Z",
      "expiresAt": "2024-01-10T00:00:00Z",
      "replacedById": "bond_uuid_1",
      "stellarContractAddress": null,
      "createdAt": "2023-01-10T09:00:00Z"
    }
  ]
}
```

**Bond Object Fields**:

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Bond unique identifier |
| `bondNumber` | string | CBP bond number |
| `policyType` | string | Bond policy type (`single_entry` or `continuous`) |
| `coverageAmount` | string | Bond coverage amount |
| `status` | string | Bond status: `active`, `pending`, `expired`, or `replaced` |
| `issuedAt` | timestamp | Bond issuance date |
| `expiresAt` | timestamp | Bond expiration date |
| `replacedById` | UUID\|null | ID of the bond that replaced this one (if applicable) |
| `stellarContractAddress` | string\|null | Stellar contract address for on-chain bonds |
| `createdAt` | timestamp | Record creation timestamp |

**Error Responses**:
- `403 Forbidden`: User not authorized to view bonds for this importer
- `404 Not Found`: Importer not found

**Notes**:
- For paginated bond listings with sorting and filtering, see `GET /importers/:id/portfolio`
- Bonds are returned in reverse chronological order (newest first)

---

## Collateral Deposit

### POST /importers/:id/deposit

Deposit funds into an importer's collateral or reserve bucket on Stellar. The deposit is processed asynchronously.

**Authorization**: Importer owner or surety admin

**KYC Requirement**: Importer must have `kyc_status` of `approved`

**AML Screening**: The importer's Stellar address is screened before processing. High-risk addresses are blocked.

**Request Body** (JSON):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amountStroops` | string | Yes | Amount to deposit in stroops (1 XLM = 10,000,000 stroops). Must be a positive integer string. |
| `bucket` | string | Yes | Target bucket: `collateral` or `reserve` |

**Bucket Semantics**:
- `collateral`: Funds are added to the importer's collateral balance, backing their customs bond requirements
- `reserve`: Funds are added to the importer's reserve balance, available for future collateral needs

**Example Request**:
```bash
curl -X POST https://api.tariffshield.com/api/v1/importers/abc123/deposit \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "amountStroops": "1000000000",
    "bucket": "collateral"
  }'
```

**Response** (202 Accepted):
```json
{
  "jobId": "job_abc123",
  "statusUrl": "/importers/abc123/tx-status/job_abc123"
}
```

**Error Responses**:
- `400 Bad Request`: Invalid input (non-numeric amount, invalid bucket)
- `403 Forbidden`: KYC not approved or AML review required
- `404 Not Found`: Importer not found

**Notes**:
- The deposit transaction is submitted asynchronously to the Stellar network
- Use the `statusUrl` to poll for transaction completion
- The on-chain account cache is invalidated after deposit to ensure fresh data on next read

---

## See Also

- [Authentication Guide](./authentication.md)
- [Error Codes Reference](./error-codes.md)
- [Rate Limits](./rate-limits.md)
- [Webhooks](../guides/webhooks.md)
