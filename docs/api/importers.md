# Importers API Documentation

## Overview

The Importers API provides endpoints for managing importer entities, their compliance documents, activity events, and operational data.

## Base URL

All endpoints are prefixed with `/api/v1/importers`

## Authentication

All endpoints require Bearer token authentication via the `Authorization` header:

```
Authorization: Bearer <your-jwt-token>
```

---

## Documents Management

### POST /importers/:id/documents

Upload compliance documents for an importer (e.g., licenses, certifications, Form 301).

**Authorization**: Importer owner or surety admin

**Request Format**: `multipart/form-data`

**Parameters**:
- `id` (path, required): Importer ID

**Request Body**:
- `document` (file, required): The document file to upload
- `documentType` (string, required): Document classification
  - Allowed values: `license`, `form_301`, `certification`, `compliance`, `other`
- `description` (string, optional): Human-readable document description
- `expiresAt` (ISO 8601 date, optional): Document expiration date

**Accepted File Types**:
- PDF (`.pdf`)
- Images: JPEG (`.jpg`, `.jpeg`), PNG (`.png`)
- Documents: DOCX (`.docx`), DOC (`.doc`)

**File Size Limits**:
- Maximum file size: 10 MB
- Files are automatically scanned for viruses before storage

**Example Request**:
```bash
curl -X POST https://api.tariffshield.com/api/v1/importers/abc123/documents \
  -H "Authorization: Bearer <token>" \
  -F "document=@license.pdf" \
  -F "documentType=license" \
  -F "description=Import License 2024" \
  -F "expiresAt=2024-12-31T23:59:59Z"
```

**Response** (201 Created):
```json
{
  "document": {
    "id": "doc_xyz789",
    "importerId": "abc123",
    "documentType": "license",
    "description": "Import License 2024",
    "filename": "license.pdf",
    "fileSize": 245760,
    "contentType": "application/pdf",
    "uploadedBy": "user_123",
    "uploadedAt": "2024-01-15T10:30:00Z",
    "expiresAt": "2024-12-31T23:59:59Z",
    "scanStatus": "clean",
    "s3Key": "encrypted-key"
  }
}
```

**Error Responses**:
- `400 Bad Request`: Invalid file type or missing required fields
- `403 Forbidden`: User not authorized to upload documents for this importer
- `413 Payload Too Large`: File exceeds 10 MB limit
- `422 Unprocessable Entity`: Virus detected in uploaded file

---

### GET /importers/:id/documents

List all documents for an importer with pagination support.

**Authorization**: Importer owner or surety admin

**Parameters**:
- `id` (path, required): Importer ID
- `limit` (query, optional): Number of documents per page (default: 20, max: 100)
- `offset` (query, optional): Number of documents to skip (default: 0)
- `documentType` (query, optional): Filter by document type
- `includeExpired` (query, optional): Include expired documents (default: false)

**Example Request**:
```bash
curl -X GET "https://api.tariffshield.com/api/v1/importers/abc123/documents?limit=10&documentType=license" \
  -H "Authorization: Bearer <token>"
```

**Response** (200 OK):
```json
{
  "documents": [
    {
      "id": "doc_xyz789",
      "importerId": "abc123",
      "documentType": "license",
      "description": "Import License 2024",
      "filename": "license.pdf",
      "fileSize": 245760,
      "contentType": "application/pdf",
      "uploadedBy": "user_123",
      "uploadedAt": "2024-01-15T10:30:00Z",
      "expiresAt": "2024-12-31T23:59:59Z",
      "scanStatus": "clean",
      "downloadUrl": "https://api.tariffshield.com/api/v1/importers/abc123/documents/doc_xyz789/download"
    }
  ],
  "pagination": {
    "total": 45,
    "limit": 10,
    "offset": 0,
    "hasMore": true
  }
}
```

**Error Responses**:
- `403 Forbidden`: User not authorized to view documents for this importer
- `404 Not Found`: Importer not found

---

### DELETE /importers/:id/documents/:docId

Delete an importer document.

**Authorization**: Importer owner or surety admin

**Deletion Behavior**: **Soft delete** — Documents are marked as deleted but retained for audit compliance. Physical deletion occurs after the regulatory retention period (7 years).

**Parameters**:
- `id` (path, required): Importer ID
- `docId` (path, required): Document ID

**Example Request**:
```bash
curl -X DELETE https://api.tariffshield.com/api/v1/importers/abc123/documents/doc_xyz789 \
  -H "Authorization: Bearer <token>"
```

**Response** (200 OK):
```json
{
  "deleted": true,
  "documentId": "doc_xyz789",
  "deletedAt": "2024-01-15T11:45:00Z",
  "deletedBy": "user_123",
  "retentionUntil": "2031-01-15T11:45:00Z"
}
```

**Error Responses**:
- `403 Forbidden`: User not authorized to delete documents for this importer
- `404 Not Found`: Document not found or already deleted
- `409 Conflict`: Document is referenced by active compliance review and cannot be deleted

**Notes**:
- Deleted documents are excluded from GET /documents responses unless `includeDeleted=true` is specified (admin only)
- Download URLs for deleted documents return 410 Gone
- Audit logs retain references to deleted documents

---

## Activity Events

### GET /importers/:id/events

Retrieve the activity/audit timeline for an importer.

**Authorization**: Importer owner or surety admin

**Parameters**:
- `id` (path, required): Importer ID
- `limit` (query, optional): Events per page (default: 50, max: 200)
- `offset` (query, optional): Events to skip (default: 0)
- `eventType` (query, optional): Filter by event type
- `startDate` (query, optional): Filter events after this date (ISO 8601)
- `endDate` (query, optional): Filter events before this date (ISO 8601)

**Event Types**:
- `importer_created`: Importer entity registered
- `document_uploaded`: Compliance document uploaded
- `document_deleted`: Document removed
- `collateral_deposited`: Funds deposited to collateral account
- `collateral_withdrawn`: Funds withdrawn from collateral
- `bond_created`: Customs bond issued
- `bond_updated`: Bond details modified
- `review_started`: Compliance review initiated
- `review_approved`: Review approved
- `review_rejected`: Review rejected
- `credit_line_granted`: Credit line extended
- `credit_line_revoked`: Credit line terminated
- `tariff_uploaded`: Tariff schedule uploaded
- `alert_triggered`: Tariff spike alert fired
- `webhook_configured`: Webhook endpoint added
- `api_key_created`: API key generated
- `api_key_revoked`: API key disabled

**Example Request**:
```bash
curl -X GET "https://api.tariffshield.com/api/v1/importers/abc123/events?limit=20&eventType=document_uploaded&startDate=2024-01-01T00:00:00Z" \
  -H "Authorization: Bearer <token>"
```

**Response** (200 OK):
```json
{
  "events": [
    {
      "id": "evt_123",
      "importerId": "abc123",
      "eventType": "document_uploaded",
      "timestamp": "2024-01-15T10:30:00Z",
      "actorId": "user_123",
      "actorEmail": "importer@example.com",
      "actorRole": "importer",
      "metadata": {
        "documentId": "doc_xyz789",
        "documentType": "license",
        "filename": "license.pdf",
        "fileSize": 245760
      },
      "ipAddress": "203.0.113.42"
    },
    {
      "id": "evt_124",
      "importerId": "abc123",
      "eventType": "collateral_deposited",
      "timestamp": "2024-01-15T11:00:00Z",
      "actorId": "user_123",
      "actorEmail": "importer@example.com",
      "actorRole": "importer",
      "metadata": {
        "amount": "50000.00",
        "currency": "USD",
        "txHash": "abc...def",
        "stellarAddress": "GABC...XYZ"
      },
      "ipAddress": "203.0.113.42"
    }
  ],
  "pagination": {
    "total": 342,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  }
}
```

**Error Responses**:
- `403 Forbidden`: User not authorized to view events for this importer
- `404 Not Found`: Importer not found

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
