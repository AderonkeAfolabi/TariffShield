# Importers API Reference

This document is the standalone API reference for the importer endpoints in `apps/api/src/routes/importers.ts`. It covers importer registration, profile/metrics reads, auto-top-up, and oracle data reconciliation.

All endpoints in this document are mounted under the `/importers` prefix and require a valid Bearer JWT (`Authorization: Bearer <token>`). Every route in this router is also wrapped by the privacy-policy and terms-of-service re-acceptance gates.

## Authentication & Roles

Two roles interact with these endpoints:

| Role | Access |
|------|--------|
| `importer` | Own record only — every `:id` route verifies `importers.user_id` matches the authenticated user |
| `surety_admin` | Any importer record — surety dashboard and operations use this role |

Ownership is enforced by `loadImporterFor` (importers.ts:327): `surety_admin` lookups query by id alone; importer lookups add `AND user_id = <authenticated user>`. A failed lookup returns `404 { "error": "not found" }` — the API does not disclose whether an id exists for a different owner.

---

## POST /importers

Register an importer. This is the onboarding entry point: it creates the importer record, generates and registers a Stellar keypair, and writes the CBP Form 301 bond record. Only `importer`-role accounts may call it.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `legalName` | string | Yes | Legal name of the importing entity. Min 1 character. Used verbatim in OFAC screening and bond validation. |
| `ein` | string | No | US Employer Identification Number. Used in OFAC screening and stored (alongside a SHA-256 hash for dedup lookups). |
| `bondId` | integer | Yes | CBP bond identifier. Must be a positive integer; passed to `register_importer` on-chain. |
| `initialRequiredCollateral` | string | Yes | Initial collateral requirement as a digit-only string in **stroops** (7 decimals, i.e. 1 XLM/USDC = 10,000,000 stroops). Must match `/^\d+$/`. |
| `businessState` | string | No | Two-letter US state code (e.g. `"CA"`). Uppercased by the server; defaults to `"CA"`. |

### Processing Pipeline

The handler runs several checks in order; the first failure wins:

1. **Role check** — `role !== "importer"` → 403.
2. **Schema validation** — Zod `CreateImporterSchema` → 400 with `details`.
3. **OFAC screening** — `screenImporterEntity(legalName, ein)`; a sanctions-list hit → 403.
4. **Duplicate check** — existing `importers` row for this `user_id` → 409.
5. **AML screening** — a freshly generated Stellar keypair is screened; `riskScore === "HIGH"` → 403.
6. **Bond validation** — `validateBondForm301` (CBP Form 301 rules: bond type `02`, principal name/EIN match) → 422 with `details`.

On success the server inserts the importer + `bond_records` row, funds the new Stellar address via Friendbot (testnet only), calls `contractClient.registerImporter` (signed by the platform admin keypair), and records the `register` contract event.

### Response — `200 OK`

```json
{
  "importer": {
    "id": "3f8a2c1e-…",
    "legalName": "Acme Imports LLC",
    "ein": "12-3456789",
    "bondId": 100234,
    "stellarAddress": "GABCD…",
    "stellarSecret": "SBEI…",
    "registeredOnChainTx": "a1b2…",
    "stellarTxUrl": "https://stellar.expert/explorer/testnet/tx/a1b2…",
    "createdAt": "2026-10-16T12:00:00.000Z"
  }
}
```

| Field | Description |
|-------|-------------|
| `id` | Importer UUID, used as `:id` on every other endpoint in this document. |
| `stellarAddress` | Public key of the newly generated Stellar account. |
| `stellarSecret` | Secret key of the new account — **returned only once** at registration. Store it client-side; it is never retrievable again. |
| `registeredOnChainTx` | Transaction hash of the on-chain `register_importer` call. |
| `stellarTxUrl` | Block-explorer URL for the registration transaction. |

### Status Codes

| Status | Condition |
|--------|-----------|
| 200 | Importer registered (note: the handler returns `res.json(...)`, i.e. 200, not 201) |
| 400 | Body fails schema validation (`details` included) |
| 403 | Caller role is not `importer`; OFAC hit; AML high-risk wallet |
| 409 | An importer is already registered for this user |
| 422 | CBP Form 301 bond validation failed (`details` included) |

### Example

```bash
curl -X POST https://api.tariffshield.example/importers \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "legalName": "Acme Imports LLC",
    "ein": "12-3456789",
    "bondId": 100234,
    "initialRequiredCollateral": "50000000000",
    "businessState": "CA"
  }'
```

---

## GET /importers/:id

Fetch a single importer's profile plus its live on-chain account state. Common data source for dashboards and integration UIs.

### Path Parameters

| Param | Type | Description |
|-------|------|-------------|
| `id` | string (uuid) | Importer ID from registration. Must be owned by the caller (unless `surety_admin`). |

### Response — `200 OK`

```json
{
  "importer": {
    "id": "3f8a2c1e-…",
    "legalName": "Acme Imports LLC",
    "ein": "12-3456789",
    "bondId": 100234,
    "stellarAddress": "GABCD…",
    "registeredOnChainTx": "a1b2…",
    "kycStatus": "approved",
    "createdAt": "2026-10-16T12:00:00.000Z"
  },
  "onChainAccount": {
    "bondId": "100234",
    "collateralBalance": "50000000000",
    "requiredCollateral": "50000000000",
    "reserveBalance": "10000000000",
    "yieldAccrued": "15000000",
    "isClawbacked": false
  }
}
```

**`importer` object:**

| Field | Description |
|-------|-------------|
| `id` | Importer UUID |
| `legalName` | Registered legal name |
| `ein` | Stored EIN (nullable) |
| `bondId` | CBP bond identifier |
| `stellarAddress` | On-chain account address |
| `registeredOnChainTx` | Registration transaction hash |
| `kycStatus` | KYC state machine value — `pending` / `in_review` / `approved` / `rejected` |
| `createdAt` | Registration timestamp (ISO 8601) |

**`onChainAccount` object** (all amounts as stroop strings):

| Field | Description |
|-------|-------------|
| `bondId` | Bond id currently set on-chain |
| `collateralBalance` | Locked collateral bucket balance |
| `requiredCollateral` | Current on-chain requirement (recomputed by each tariff upload) |
| `reserveBalance` | Liquid reserve bucket balance — source for auto-top-up |
| `yieldAccrued` | Yield accrued to date on both buckets |
| `isClawbacked` | Whether the surety admin has clawed back this account |

### Caching

On-chain state is served cache-aside from Redis (`getCachedOnChainAccount`): entries ≤ 30 s old are served directly; stale entries trigger a live Soroban RPC read and repopulate the cache. Tariff uploads, deposits, withdrawals, and auto-top-ups invalidate the cache immediately so subsequent reads reflect fresh state.

### Status Codes

| Status | Condition |
|--------|-----------|
| 200 | Importer found and accessible to caller |
| 404 | No importer with that id, or owned by a different user |

---

## GET /importers/:id/metrics

Fetch the pre-computed metrics row for a single importer, sourced from the `importer_metrics` materialized view (refreshed on a 5-minute schedule by `jobs/refresh-importer-metrics.ts`).

### Path Parameters

| Param | Type | Description |
|-------|------|-------------|
| `id` | string (uuid) | Importer ID. Ownership enforced as on `GET /importers/:id`. |

### Response — `200 OK`

```json
{
  "metrics": {
    "importer_id": "3f8a2c1e-…",
    "legal_name": "Acme Imports LLC",
    "stellar_address": "GABCD…",
    "required_collateral": "50000000000",
    "current_balance": "50000000000",
    "annual_duty_total": "75000000",
    "coverage_ratio": 1.0,
    "refreshed_at": "2026-10-16T11:55:00.000Z"
  }
}
```

| Field | Description |
|-------|-------------|
| `importer_id` | Importer UUID (view primary key) |
| `legal_name` | Registered legal name |
| `stellar_address` | On-chain account address |
| `required_collateral` | Required collateral from the importer's latest tariff upload (stroops; `0` if never uploaded) |
| `current_balance` | Net of all deposit/withdrawal contract events: Σ deposits − Σ withdrawals |
| `annual_duty_total` | Annual duties+taxes+fees from the latest tariff upload |
| `coverage_ratio` | `current_balance / required_collateral`, rounded to 4 decimals; `null` when no requirement is set |
| `refreshed_at` | When the materialized view was last refreshed |

### Status Codes

| Status | Condition |
|--------|-----------|
| 200 | Metrics row returned |
| 404 | `"not found"` — importer missing/not owned; `"metrics not found for importer"` — no row in the materialized view yet |
| 500 | Database query failure |

---

## POST /importers/:id/auto-top-up

Trigger an automatic collateral top-up. When a tariff spike raises `required_collateral` above the `collateral_balance`, this endpoint enqueues an on-chain `auto_top_up` transaction that moves `min(shortfall, reserve)` from the importer's **reserve** bucket into its **collateral** bucket.

### Interaction with Collateral Thresholds

- **Trigger condition:** `required_collateral > collateral_balance` (a shortfall). The on-chain contract computes the transferable amount as `min(required − collateral, reserve_balance)` — the top-up never pulls more than the shortfall and never exceeds the reserve.
- **Reserve prerequisite:** the importer must first fund the reserve bucket via `POST /importers/:id/deposit` with `bucket: "reserve"`. An empty reserve means the top-up moves nothing.
- **After a tariff upload:** `POST /importers/:id/upload-tariff-csv` recomputes and updates `required_collateral` on-chain. If the new requirement exceeds the current collateral balance, call this endpoint to close the gap automatically rather than waiting for a manual deposit.
- **KYC gate:** blocked until `kyc_status === "approved"` (importers.ts:812), matching the manual deposit rule.

### Request Body

None.

### Response — `202 Accepted`

The transaction is submitted asynchronously through the tx queue; poll `statusUrl` for the result.

```json
{
  "jobId": "9c1e4b7a-…",
  "statusUrl": "/importers/3f8a2c1e-…/tx-status/9c1e4b7a-…"
}
```

The on-chain transaction is signed by the platform key (`platformKey: true` in the queue job), so the importer does not need to submit a signature.

### Status Codes

| Status | Condition |
|--------|-----------|
| 202 | Top-up transaction enqueued |
| 403 | `kyc_status !== "approved"` (response includes `kycStatus`) |
| 404 | Importer not found or not owned by caller |

### Example

```bash
curl -X POST https://api.tariffshield.example/importers/3f8a2c1e-…/auto-top-up \
  -H "Authorization: Bearer $JWT"
```

---

## POST /importers/:id/verify-oracle-data

Reconcile the on-chain `required_collateral` against the importer's latest tariff upload. This is the verification step of the oracle rate-update flow: after `POST /importers/:id/upload-tariff-csv` recomputes the requirement and writes it on-chain, this endpoint independently re-derives the expected value from the stored tariff data and reports whether the two agree.

### Relationship to the Oracle Rate-Update Flow

1. **Rate update:** `POST /importers/:id/upload-tariff-csv` computes `required = annual_duty_total × 10% × 50%` (CBP continuous-bond rule of thumb: 50% of a bond face value equal to 10% of annual duties) in stroops and calls `set_required_collateral` on-chain via the platform admin.
2. **Verification:** this endpoint re-derives the same formula from the latest `tariff_uploads` row, reads the live on-chain value via `getRequiredCollateralOnChain`, and compares.
3. **Mismatch handling:** when the deviation exceeds 1%, a `reconciliation_failure` row is written to `oracle_alerts` (deduplicated by `ON CONFLICT DO NOTHING`). Integrations can poll this endpoint to confirm that an oracle-sourced rate change landed on-chain correctly.

### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `as_of_date` | string (ISO 8601 datetime) | No | Reconcile against the latest tariff upload created at or before this timestamp. Omit to use the most recent upload. |

### Response — `200 OK`

```json
{
  "computed": "50000000000",
  "on_chain": "50000000000",
  "match": true,
  "deviation_pct": 0,
  "csv_hash": "a3f1…",
  "collateral_timestamp": "2026-10-16T12:00:00.000Z"
}
```

| Field | Description |
|-------|-------------|
| `computed` | Expected required collateral re-derived from the tariff upload (stroops) |
| `on_chain` | Live on-chain `required_collateral` (stroops) |
| `match` | `true` when `deviation_pct ≤ 1.0` |
| `deviation_pct` | `|computed − on_chain| / on_chain × 100` (0 when both are 0; 100 when on-chain is 0 but computed is not) |
| `csv_hash` | SHA-256 fingerprint of the latest tariff upload (`filename:annual_duty_total`) — stable identifier for the source data |
| `collateral_timestamp` | When the reconciled tariff upload was created |

### Status Codes

| Status | Condition |
|--------|-----------|
| 200 | Reconciliation completed (check `match` / `deviation_pct` for the result) |
| 400 | Body fails `VerifyOracleSchema` validation (e.g. `as_of_date` not ISO 8601) |
| 404 | `"not found"` — importer missing/not owned; `"no tariff CSV data found for this importer"` — importer has never uploaded a tariff CSV |

### Example

```bash
curl -X POST https://api.tariffshield.example/importers/3f8a2c1e-…/verify-oracle-data \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"as_of_date": "2026-10-16T00:00:00Z"}'
```
