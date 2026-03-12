# Delegation Flow Example

This example shows a complete delegation lifecycle: Alice delegates rights to Bob at an SP, Bob acts on Alice's behalf, and the SP validates the delegation.

**Actors:**
- Alice (`alice@example.com`) — Delegator
- Bob (`bob@example.com`) — Delegate
- IdP: `https://id.example.com`
- SP: `app.example.com`

## Step 1: Alice Creates Delegation

Alice (authenticated via session) creates a delegation for Bob.

**Request:**

```http
POST /api/delegations HTTP/1.1
Host: id.example.com
Cookie: session=<alice-session>
Content-Type: application/json

{
  "delegate": "bob@example.com",
  "audience": "app.example.com",
  "scopes": ["read", "write"],
  "grant_type": "timed",
  "duration": 86400
}
```

**Response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
  "type": "delegation",
  "request": {
    "requester": "alice@example.com",
    "target": "app.example.com",
    "grant_type": "timed",
    "duration": 86400,
    "delegator": "alice@example.com",
    "delegate": "bob@example.com",
    "audience": "app.example.com",
    "scopes": ["read", "write"]
  },
  "status": "approved",
  "decided_by": "alice@example.com",
  "created_at": 1710244800,
  "decided_at": 1710244800,
  "expires_at": 1710331200
}
```

Note: The delegation is auto-approved since Alice (the delegator) is creating it.

## Step 2: Bob Authenticates at the IdP

Bob authenticates normally (via WebAuthn or Ed25519). The IdP issues an assertion with delegation claims.

### Bob's Assertion JWT (Decoded Payload)

```json
{
  "iss": "https://id.example.com",
  "sub": "alice@example.com",
  "aud": "app.example.com",
  "iat": 1710245000,
  "exp": 1710245300,
  "nonce": "delegation-nonce-123",
  "act": {
    "sub": "bob@example.com"
  },
  "delegate": {
    "sub": "bob@example.com",
    "act": "agent",
    "grant_id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890"
  },
  "delegation_grant": "d1e2f3a4-b5c6-7890-abcd-ef1234567890"
}
```

Key observations:
- `sub` is **Alice** (the delegator — the person being acted on behalf of)
- `act.sub` is **Bob** (the actual actor)
- `delegation_grant` references the delegation grant ID

## Step 3: Bob Accesses the SP

Bob presents the delegation JWT to the SP.

**Request:**

```http
GET /api/data HTTP/1.1
Host: app.example.com
Authorization: Bearer <delegation-jwt>
```

## Step 4: SP Validates Delegation

The SP performs standard assertion validation plus delegation-specific checks.

### Standard Validation

| Check | Result |
|-------|--------|
| Signature (JWKS) | PASS |
| `iss` matches DNS-discovered IdP | PASS |
| `aud` matches own client_id | PASS |
| `exp` in the future | PASS |

### Delegation Validation

The SP detects `act` is an object (not a string), indicating this is a delegated action.

**Optional: Validate delegation at IdP**

```http
POST /api/delegations/d1e2f3a4-b5c6-7890-abcd-ef1234567890/validate HTTP/1.1
Host: id.example.com
Content-Type: application/json

{
  "delegate": "bob@example.com",
  "audience": "app.example.com"
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "valid": true,
  "delegation": {
    "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
    "type": "delegation",
    "status": "approved",
    ...
  },
  "scopes": ["read", "write"]
}
```

### SP Applies Scope Restrictions

The SP checks that the requested action (`GET /api/data`) falls within the delegation's scopes (`["read", "write"]`). Since reading data is within the `read` scope, the request is allowed.

**Response to Bob:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [ ... ]
}
```

## Step 5: Alice Lists Her Delegations

**Request:**

```http
GET /api/delegations?role=delegator HTTP/1.1
Host: id.example.com
Cookie: session=<alice-session>
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

[
  {
    "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
    "type": "delegation",
    "request": {
      "delegator": "alice@example.com",
      "delegate": "bob@example.com",
      "audience": "app.example.com",
      "scopes": ["read", "write"],
      ...
    },
    "status": "approved",
    "expires_at": 1710331200,
    ...
  }
]
```

## Step 6: Alice Revokes Delegation

**Request:**

```http
DELETE /api/delegations/d1e2f3a4-b5c6-7890-abcd-ef1234567890 HTTP/1.1
Host: id.example.com
Cookie: session=<alice-session>
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
  "type": "delegation",
  "status": "revoked",
  ...
}
```

## Step 7: Post-Revocation Validation Fails

```http
POST /api/delegations/d1e2f3a4-b5c6-7890-abcd-ef1234567890/validate HTTP/1.1
Host: id.example.com
Content-Type: application/json

{
  "delegate": "bob@example.com",
  "audience": "app.example.com"
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "valid": false,
  "error": "Delegation grant is not approved: revoked"
}
```
