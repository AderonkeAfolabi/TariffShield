# Authentication API

Reference documentation for the authentication endpoints in the TariffShield API.

---

## `GET /auth/me`

Returns the currently authenticated user's profile.

### Authentication

Requires a valid Bearer token obtained from `POST /auth/login`, `POST /auth/signup`, or the SAML callback.

```
Authorization: Bearer <token>
```

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `user.id` | `string` (UUID) | Unique user identifier |
| `user.email` | `string` | Email address associated with the account |
| `user.role` | `"importer"` \| `"surety_admin"` | Role assigned to this account |
| `user.sessionId` | `string` (optional) | Active session identifier (used for session management and logout). Present on all tokens issued after session management was deployed. |

### Example response

```json
{
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "admin@example.com",
    "role": "surety_admin",
    "sessionId": "ses_abc123def456"
  }
}
```

### Errors

| Status | Condition |
|--------|-----------|
| 401 | Missing or invalid Bearer token |
| 401 | Session expired or revoked |
