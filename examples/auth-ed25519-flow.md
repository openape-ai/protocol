# Ed25519 Challenge-Response Authentication Flow Example

This example shows the complete Ed25519 authentication flow for an automated agent.

**Actors:**
- Agent: `deploy-bot@example.com` (registered at the IdP with an Ed25519 public key)
- IdP: `https://id.example.com`

**Prerequisites:**
- The agent has an Ed25519 key pair.
- The agent's public key is registered with the IdP.
- The IdP's discovery document includes `ddisa_agent_challenge_endpoint` and `ddisa_agent_authenticate_endpoint`.

## Step 1: Request Challenge

**Request:**

```http
POST /api/agent/challenge HTTP/1.1
Host: id.example.com
Content-Type: application/json

{
  "agent_id": "deploy-bot@example.com"
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "challenge": "a7f3b2c1-9d8e-4f5a-b6c7-d8e9f0a1b2c3"
}
```

## Step 2: Sign Challenge

The agent signs the challenge string with its Ed25519 private key:

```
Input:     "a7f3b2c1-9d8e-4f5a-b6c7-d8e9f0a1b2c3"  (exact challenge string)
Algorithm: Ed25519
Output:    Base64-encoded signature
```

## Step 3: Authenticate

**Request:**

```http
POST /api/agent/authenticate HTTP/1.1
Host: id.example.com
Content-Type: application/json

{
  "agent_id": "deploy-bot@example.com",
  "challenge": "a7f3b2c1-9d8e-4f5a-b6c7-d8e9f0a1b2c3",
  "signature": "MEUCIQC7cYt4K6P7PkDOL3jGqr3xSv8JkVfBqM7LHXE8bG+vQwIgZzYp..."
}
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImlkcC1rZXktMjAyNi0wMSJ9.eyJpc3MiOiJodHRwczovL2lkLmV4YW1wbGUuY29tIiwic3ViIjoiZGVwbG95LWJvdEBleGFtcGxlLmNvbSIsImlhdCI6MTcxMDI0NDgwMCwiZXhwIjoxNzEwMjQ4NDAwfQ.signature",
  "agent_id": "b2c3d4e5-f6a7-8901-bcde-f23456789012",
  "email": "deploy-bot@example.com",
  "name": "Deploy Bot",
  "expires_in": 3600
}
```

## Step 4: Decoded Token

### JWT Header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "idp-key-2026-01"
}
```

### JWT Payload

```json
{
  "iss": "https://id.example.com",
  "sub": "deploy-bot@example.com",
  "iat": 1710244800,
  "exp": 1710248400
}
```

## Step 5: Use Token

The agent uses the token as a Bearer token for subsequent API calls:

```http
POST /api/grants HTTP/1.1
Host: id.example.com
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
Content-Type: application/json

{
  "target": "production-server",
  "grant_type": "once",
  "command": ["deploy", "--env", "production"],
  "reason": "Scheduled production deployment"
}
```

## Error Cases

### Agent Not Found

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "statusCode": 404,
  "statusMessage": "Agent not found or inactive"
}
```

### Invalid/Expired Challenge

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "statusCode": 401,
  "statusMessage": "Invalid, expired, or already used challenge"
}
```

### Invalid Signature

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "statusCode": 401,
  "statusMessage": "Invalid signature"
}
```
