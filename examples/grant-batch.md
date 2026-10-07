# Grant Batch Example

This example shows three `once` grants submitted as one request batch and
decided together: two are approved, one is denied. Every member remains an
independent grant; `batch` only groups them for display.

**Actors:**
- Agent: `archive-bot@example.com` (requester)
- Human: `alice@example.com` (approver)
- IdP Grants API: `https://id.example.com/api/grants`

## Step 1: Create the members

The agent creates one grant per item with the same `batch.id`. Only the first
request is shown; the other two differ in `command` and `summary`.

```http
POST /api/grants HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
Content-Type: application/json

{
  "requester": "archive-bot@example.com",
  "target_host": "mail.example.com",
  "audience": "mail-archive",
  "grant_type": "once",
  "permissions": ["mail.archive"],
  "command": ["mail-archive", "move", "{\"message\":\"m-101\",\"digest\":\"SHA-256:5d1c…\"}"],
  "summary": { "text": "\"Newsletter XY\" <news@xy.example> – What's new" },
  "waits_until": 1791345600,
  "batch": { "id": "b-20261007-0900", "title": "Newsletters, October 7", "size": 3 }
}
```

## Step 2: Poll the batch

```http
GET /api/grants?requester=archive-bot%40example.com&batch=b-20261007-0900 HTTP/1.1
Host: id.example.com
Authorization: Bearer <agent-token>
```

```json
{
  "data": [
    { "id": "g-3", "status": "pending", "request": { "batch": { "id": "b-20261007-0900", "title": "Newsletters, October 7", "size": 3 } } },
    { "id": "g-2", "status": "pending", "request": { "batch": { "id": "b-20261007-0900", "title": "Newsletters, October 7", "size": 3 } } },
    { "id": "g-1", "status": "pending", "request": { "batch": { "id": "b-20261007-0900", "title": "Newsletters, October 7", "size": 3 } } }
  ],
  "pagination": { "cursor": null, "has_more": false }
}
```

## Step 3: Decide the batch

The approval UI shows one card with a checkbox per member. Alice keeps two
members selected and confirms; the UI denies the unselected member.

```http
POST /api/grants/batch HTTP/1.1
Host: id.example.com
Cookie: <alice-session>
Content-Type: application/json

{
  "operations": [
    { "id": "g-1", "action": "approve" },
    { "id": "g-2", "action": "approve" },
    { "id": "g-3", "action": "deny" }
  ]
}
```

```json
{
  "results": [
    { "id": "g-1", "status": "approved", "success": true },
    { "id": "g-2", "status": "approved", "success": true },
    { "id": "g-3", "status": "denied", "success": true }
  ]
}
```

## Step 4: Use the approved members

For each approved member the agent obtains an AuthZ-JWT
(`POST /api/grants/g-1/token`) and consumes it (`POST /api/grants/g-1/consume`)
before acting, exactly as for any other `once` grant. The denied member is
never executed.
