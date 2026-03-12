# Grant Lifecycle Example

This example shows the complete lifecycle of a `once` grant: create, poll, approve, get token, consume.

**Actors:**
- Agent: `deploy-bot@example.com` (requester)
- Human: `alice@example.com` (approver)
- IdP Grants API: `https://id.example.com/api/grants`

## Step 1: Create Grant

The agent creates a grant request.

**Request:**

```http
POST /api/grants HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
Content-Type: application/json

{
  "target": "production-server",
  "grant_type": "once",
  "command": ["deploy", "--env", "production", "--tag", "v2.1.0"],
  "cmd_hash": "SHA-256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "reason": "Deploy v2.1.0 to production"
}
```

**Response:**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "request": {
    "requester": "deploy-bot@example.com",
    "target": "production-server",
    "grant_type": "once",
    "command": ["deploy", "--env", "production", "--tag", "v2.1.0"],
    "cmd_hash": "SHA-256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
    "reason": "Deploy v2.1.0 to production"
  },
  "status": "pending",
  "created_at": 1710244800
}
```

## Step 2: Poll for Status

The agent polls for status updates.

**Request:**

```http
GET /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890 HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
```

**Response (still pending):**

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v1-pending"

{
  "id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "request": { ... },
  "status": "pending",
  "created_at": 1710244800
}
```

**Subsequent poll with ETag:**

```http
GET /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890 HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
If-None-Match: "v1-pending"
```

**Response (not modified):**

```http
HTTP/1.1 304 Not Modified
```

## Step 3: Approve Grant

The human approver approves the grant.

**Request:**

```http
POST /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890/approve HTTP/1.1
Host: id.example.com
Cookie: session=<session-cookie>
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "request": {
    "requester": "deploy-bot@example.com",
    "target": "production-server",
    "grant_type": "once",
    "command": ["deploy", "--env", "production", "--tag", "v2.1.0"],
    "cmd_hash": "SHA-256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
    "reason": "Deploy v2.1.0 to production"
  },
  "status": "approved",
  "decided_by": "alice@example.com",
  "created_at": 1710244800,
  "decided_at": 1710244830
}
```

## Step 4: Poll Detects Approval

**Request:**

```http
GET /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890 HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
If-None-Match: "v1-pending"
```

**Response (changed — approved):**

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "v2-approved"

{
  "id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "status": "approved",
  "decided_by": "alice@example.com",
  "decided_at": 1710244830,
  ...
}
```

## Step 5: Get Authorization JWT

**Request:**

```http
POST /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890/token HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "authzJWT": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImlkcC1rZXktMjAyNi0wMSJ9.eyJpc3MiOiJodHRwczovL2lkLmV4YW1wbGUuY29tIiwic3ViIjoiZGVwbG95LWJvdEBleGFtcGxlLmNvbSIsImF1ZCI6InByb2R1Y3Rpb24tc2VydmVyIiwiaWF0IjoxNzEwMjQ0ODQwLCJleHAiOjE3MTAyNDUxNDAsImp0aSI6IjEyMzQ1Njc4LTkwYWItY2RlZi0xMjM0LTU2Nzg5MGFiY2RlZiIsImdyYW50X2lkIjoiZzFhMmIzYzQtZDVlNi03ODkwLWFiY2QtZWYxMjM0NTY3ODkwIiwiZ3JhbnRfdHlwZSI6Im9uY2UiLCJjbWRfaGFzaCI6IlNIQS0yNTY6YTFiMmMzZDRlNWY2YTdiOGM5ZDBlMWYyYTNiNGM1ZDZlN2Y4YTliMGMxZDJlM2Y0YTViNmM3ZDhlOWYwYTFiMiIsImRlY2lkZWRfYnkiOiJhbGljZUBleGFtcGxlLmNvbSJ9.signature",
  "grant": { ... }
}
```

### Decoded AuthZ-JWT Payload

```json
{
  "iss": "https://id.example.com",
  "sub": "deploy-bot@example.com",
  "aud": "production-server",
  "iat": 1710244840,
  "exp": 1710245140,
  "jti": "12345678-90ab-cdef-1234-567890abcdef",
  "grant_id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
  "grant_type": "once",
  "approval": "once",
  "cmd_hash": "SHA-256:a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
  "command": ["deploy", "--env", "production", "--tag", "v2.1.0"],
  "decided_by": "alice@example.com"
}
```

## Step 6: Consume Grant

The executor verifies and consumes the grant before running the command.

**Request:**

```http
POST /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890/consume HTTP/1.1
Host: id.example.com
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "consumed",
  "grant": {
    "id": "g1a2b3c4-d5e6-7890-abcd-ef1234567890",
    "status": "used",
    "used_at": 1710244850,
    ...
  }
}
```

## Step 7: Attempting to Re-consume (Fails)

**Request:**

```http
POST /api/grants/g1a2b3c4-d5e6-7890-abcd-ef1234567890/consume HTTP/1.1
Host: id.example.com
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "error": "already_consumed",
  "status": "used"
}
```
