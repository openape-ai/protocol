# OpenAPE Grants Protocol

**Version:** 1.0-draft
**Status:** Draft
**Date:** 2026-03-12
**Author:** Patrick Hofmann (Delta Mind GmbH)

## Abstract

The OpenAPE Grants Protocol defines a grant-based authorization mechanism for OIDC-compatible Identity Providers. It enables Service Providers and agents to request, manage, and consume fine-grained permissions through a REST API. This protocol is independent of DDISA — it works with any OIDC-compliant IdP that implements the grants endpoints.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Discovery](#2-discovery)
3. [Data Model](#3-data-model)
4. [REST API](#4-rest-api)
5. [Pagination](#5-pagination)
6. [Authorization JWT](#6-authorization-jwt)
7. [Polling](#7-polling)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Error Responses](#9-error-responses)

---

## 1. Introduction

### 1.1 Purpose

The Grants Protocol provides a structured mechanism for requesting and managing permissions between parties in an identity ecosystem. A *grant* represents a request from one party (the requester) to perform an action at a target resource, which must be approved by an authorized decision-maker.

### 1.2 Independence from DDISA

This protocol is designed to work with **any OIDC-compliant Identity Provider**. While it integrates naturally with DDISA-based IdPs, DDISA DNS discovery is NOT required. Any IdP that publishes the required endpoints in its OIDC Discovery document can implement the Grants Protocol.

### 1.3 Grant Lifecycle

```
  Requester              IdP (Grants API)           Approver
     |                        |                        |
     |  1. POST /grants       |                        |
     |  (create request)      |                        |
     |----------------------->|                        |
     |  2. 201 Created        |                        |
     |  (status: pending)     |                        |
     |<-----------------------|                        |
     |                        |                        |
     |  3. GET /grants/{id}   |                        |
     |  (poll for status)     |                        |
     |----------------------->|                        |
     |  4. 200 OK             |                        |
     |  (status: pending)     |                        |
     |<-----------------------|                        |
     |                        |                        |
     |                        |  5. POST /{id}/approve |
     |                        |<-----------------------|
     |                        |  6. 200 OK             |
     |                        |  (status: approved)    |
     |                        |----------------------->|
     |                        |                        |
     |  7. GET /grants/{id}   |                        |
     |  (poll: approved!)     |                        |
     |----------------------->|                        |
     |  8. 200 OK             |                        |
     |  (status: approved)    |                        |
     |<-----------------------|                        |
     |                        |                        |
     |  9. POST /{id}/token   |                        |
     |  (get AuthZ-JWT)       |                        |
     |----------------------->|                        |
     | 10. 200 OK             |                        |
     |  (authzJWT)            |                        |
     |<-----------------------|                        |
     |                        |                        |
     | 11. POST /{id}/consume |                        |
     |  (verify + consume)    |                        |
     |----------------------->|                        |
     | 12. 200 OK             |                        |
     |  (status: consumed)    |                        |
     |<-----------------------|                        |
```

---

## 2. Discovery

An IdP that supports the Grants Protocol MUST advertise its support via the OIDC Discovery document (`/.well-known/openid-configuration`).

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `openape_grants_endpoint` | REQUIRED | string | Base URL for the Grants REST API. All API paths in [Section 4](#4-rest-api) are relative to this URL. |
| `openape_grant_types_supported` | REQUIRED | string[] | Supported grant types. MUST include at least one of: `"once"`, `"timed"`, `"always"`. |
| `openape_grant_categories_supported` | OPTIONAL | string[] | Supported grant categories. Values: `"command"`, `"delegation"`. Default: `["command"]`. |

**Example:**

```json
{
  "openape_grants_endpoint": "https://id.example.com/api/grants",
  "openape_grant_types_supported": ["once", "timed", "always"],
  "openape_grant_categories_supported": ["command", "delegation"]
}
```

---

## 3. Data Model

### 3.1 Grant Object

A grant represents a permission request and its lifecycle state.

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `id` | REQUIRED | string | Unique grant identifier (UUID v4). |
| `type` | OPTIONAL | string | Grant category. One of: `"command"`, `"delegation"`. Default: `"command"`. |
| `request` | REQUIRED | object | The grant request details (see [Section 3.4](#34-grant-request)). |
| `status` | REQUIRED | string | Current grant status (see [Section 3.3](#33-grant-status)). |
| `decided_by` | OPTIONAL | string | Identifier of the user who approved or denied the grant. |
| `created_at` | REQUIRED | number | Unix timestamp (seconds) when the grant was created. |
| `decided_at` | OPTIONAL | number | Unix timestamp (seconds) when the decision was made. |
| `expires_at` | OPTIONAL | number | Unix timestamp (seconds) when the grant expires. Set for `timed` grants on approval. |
| `used_at` | OPTIONAL | number | Unix timestamp (seconds) when the grant was consumed. Set for `once` grants on use. |

### 3.2 Grant Types

| Type | Description | Expiration |
|------|-------------|------------|
| `once` | Single-use grant. Transitions to `used` after consumption. | Marked `used` after single use. |
| `timed` | Time-limited grant. Valid from approval until `expires_at`. | Transitions to `expired` after `expires_at`. |
| `always` | Permanent grant. Valid until explicitly revoked. | No automatic expiration. |

### 3.3 Grant Status

| Status | Description | Terminal |
|--------|-------------|----------|
| `pending` | Awaiting approval or denial. | No |
| `approved` | Approved by an authorized user. Active and usable. | No |
| `denied` | Denied by an authorized user. | Yes |
| `revoked` | Revoked after approval. | Yes |
| `expired` | Time-limited grant past its `expires_at`. | Yes |
| `used` | Single-use grant that has been consumed. | Yes |

**State Machine:**

```
                 +--> denied
                 |
 pending --------+--> approved --+--> revoked
                                 |
                                 +--> expired  (auto, timed only)
                                 |
                                 +--> used     (auto, once only)
```

### 3.4 Grant Request

The `request` object describes what is being requested.

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `requester` | REQUIRED | string | Identifier of the requesting entity (email or agent ID). |
| `target` | REQUIRED | string | Target system or resource identifier. |
| `grant_type` | REQUIRED | string | One of: `"once"`, `"timed"`, `"always"`. |
| `permissions` | OPTIONAL | string[] | Array of requested permission identifiers. |
| `command` | OPTIONAL | string[] | Plaintext command array (for display in approval UI). |
| `cmd_hash` | OPTIONAL | string | SHA-256 hash of the command for verification. Format: `SHA-256:<hex-encoded-hash>`. Input: the JSON-serialized `command` array. |
| `duration` | CONDITIONAL | number | Duration in seconds. REQUIRED when `grant_type` is `"timed"`. |
| `reason` | OPTIONAL | string | Human-readable reason for the request. |
| `delegator` | OPTIONAL | string | Who is being acted on behalf of (delegation grants only). |
| `delegate` | OPTIONAL | string | Who is allowed to act (delegation grants only). |
| `audience` | OPTIONAL | string | At which SP the delegation is valid (delegation grants only). `"*"` for any SP. |
| `scopes` | OPTIONAL | string[] | Allowed actions under the delegation (delegation grants only). |

### 3.5 JSON Schema Reference

Machine-readable schemas for the Grant data model are available in [schemas/grant.json](schemas/grant.json) and [schemas/grant-request.json](schemas/grant-request.json).

---

## 4. REST API

All endpoints are relative to `{openape_grants_endpoint}` as advertised in the OIDC Discovery document. All requests MUST include a valid Bearer token in the `Authorization` header (see [Section 8](#8-authentication--authorization)).

### 4.1 Create Grant

```
POST /
```

Creates a new grant with status `pending`.

**Request Body:** Grant Request object (see [Section 3.4](#34-grant-request)).

**Validation Rules:**

- `requester`, `target`, and `grant_type` are REQUIRED.
- `grant_type` MUST be one of the values in `openape_grant_types_supported`.
- If `grant_type` is `"timed"`, `duration` MUST be present and positive.
- If the request is authenticated via an agent token, the `requester` field MUST be set to the authenticated agent's identity.

**Response:** `201 Created` with the full Grant object.

### 4.2 List Grants

```
GET /?role=<role>&status=<status>&type=<type>
```

Returns grants visible to the authenticated user.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `requester` | string | Filter by requester identifier. |
| `role` | string | Filter by the authenticated user's role: `"requester"`, `"approver"`, `"admin"`. |
| `status` | string | Filter by grant status. |
| `type` | string | Filter by grant category (`"command"`, `"delegation"`). |
| `limit` | number | Maximum number of results. Default: `20`. Maximum: `100`. |
| `cursor` | string | Cursor for pagination (see [Section 5](#5-pagination)). |

**Response:** `200 OK` with an array of Grant objects. See [Section 5](#5-pagination) for pagination details.

**Visibility Rules:**

- **Admin:** Sees all grants.
- **Approver:** Sees grants where they are the target or where they own/approve the requesting agent.
- **Requester:** Sees their own grants.
- **Unauthenticated (agent token):** Sees pending grants.

### 4.3 Get Grant

```
GET /{id}
```

Returns a single grant by ID.

**Response:** `200 OK` with the Grant object.

**Headers:**

- The response SHOULD include an `ETag` header for conditional polling (see [Section 7](#7-polling)).

**Auto-expiration:** If the grant is `approved`, has type `timed`, and `expires_at` is in the past, the server MUST transition the grant to `expired` before returning it.

### 4.4 Approve Grant

```
POST /{id}/approve
```

Approves a pending grant.

**Preconditions:**

- Grant status MUST be `pending`.
- Authenticated user MUST have approval authority.

**Side Effects:**

- Sets `status` to `approved`, `decided_by` to the approver, `decided_at` to current time.
- For `timed` grants: sets `expires_at` to `decided_at + request.duration`.

**Response:** `200 OK` with the updated Grant object. The response MAY include an `authz_jwt` field containing a pre-issued Authorization JWT.

### 4.5 Deny Grant

```
POST /{id}/deny
```

Denies a pending grant.

**Preconditions:**

- Grant status MUST be `pending`.
- Authenticated user MUST have denial authority.

**Side Effects:**

- Sets `status` to `denied`, `decided_by` to the denier, `decided_at` to current time.

**Response:** `200 OK` with the updated Grant object.

### 4.6 Revoke Grant

```
POST /{id}/revoke
```

Revokes an approved grant.

**Preconditions:**

- Grant status MUST be `approved`.

**Side Effects:**

- Sets `status` to `revoked`.

**Response:** `200 OK` with the updated Grant object.

### 4.7 Get Authorization JWT

```
POST /{id}/token
```

Issues an Authorization JWT (AuthZ-JWT) for an approved grant (see [Section 6](#6-authorization-jwt)).

**Preconditions:**

- Grant status MUST be `approved`.
- Authenticated agent MUST be the grant's requester.

**Response:**

```json
{
  "authzJWT": "<JWT>",
  "grant": { ... }
}
```

### 4.8 Consume Grant

```
POST /{id}/consume
```

Verifies an AuthZ-JWT and consumes the grant (for `once` grants). Used by executors before performing the authorized action.

**Authorization:** `Bearer <authz-jwt>` — The Authorization JWT issued in [Section 4.7](#47-get-authorization-jwt).

**Validation:**

1. Verify the AuthZ-JWT signature.
2. Verify `grant_id` in the JWT matches the URL parameter `{id}`.
3. Look up the grant and check its current status.

**Response by Grant Status:**

| Grant Status | HTTP Status | Response |
|-------------|-------------|----------|
| `approved` + `once` | 200 | `{ "status": "consumed", "grant": {...} }` — Grant transitions to `used`. |
| `approved` + `timed`/`always` | 200 | `{ "status": "valid", "grant": {...} }` — Grant remains `approved`. |
| `used` | 200 | `{ "error": "already_consumed", "status": "used" }` |
| `revoked` | 200 | `{ "error": "revoked", "status": "revoked" }` |
| `denied` | 200 | `{ "error": "denied", "status": "denied" }` |
| `expired` | 200 | `{ "error": "expired", "status": "expired" }` |
| `pending` | 200 | `{ "error": "not_approved", "status": "pending" }` |

> **Note:** The consume endpoint returns `200` for all grant statuses (including invalid ones) to allow the caller to distinguish between error conditions. The `error` field indicates the specific reason for failure.

### 4.9 Verify Token

```
POST /verify
```

Verifies an AuthZ-JWT without consuming the grant.

**Request Body:**

```json
{
  "token": "<authz-jwt>"
}
```

**Response:**

```json
{
  "valid": true,
  "claims": { ... }
}
```

Or on failure:

```json
{
  "valid": false,
  "error": "reason"
}
```

### 4.10 Batch Operations

```
POST /batch
```

Performs multiple approve/deny/revoke operations in a single request.

**Request Body:**

```json
{
  "operations": [
    { "id": "grant-id-1", "action": "approve" },
    { "id": "grant-id-2", "action": "deny" },
    { "id": "grant-id-3", "action": "revoke" }
  ]
}
```

**Response:**

```json
{
  "results": [
    { "id": "grant-id-1", "status": "approved", "success": true },
    { "id": "grant-id-2", "status": "denied", "success": true },
    { "id": "grant-id-3", "error": "Grant is not approved", "success": false }
  ]
}
```

The batch endpoint MUST process all operations and return results for each. Partial failures MUST NOT cause the entire batch to fail.

---

## 5. Pagination

List endpoints MUST support cursor-based pagination.

### 5.1 Request Parameters

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `limit` | number | 20 | 100 | Number of results per page. |
| `cursor` | string | — | — | Opaque cursor from a previous response. |

### 5.2 Response Format

Paginated responses MUST include pagination metadata:

```json
{
  "data": [ ... ],
  "pagination": {
    "cursor": "next-page-cursor-or-null",
    "has_more": true
  }
}
```

- `cursor`: Opaque string for the next page. `null` if no more results.
- `has_more`: Boolean indicating whether more results exist.

### 5.3 Ordering

Results MUST be ordered by `created_at` descending (newest first) by default.

---

## 6. Authorization JWT

An Authorization JWT (AuthZ-JWT) is a short-lived token that authorizes a specific action based on an approved grant. It is issued by the IdP and consumed by the target system.

### 6.1 Claims

| Claim | Status | Type | Description |
|-------|--------|------|-------------|
| `iss` | REQUIRED | string | Issuer (IdP URL). |
| `sub` | REQUIRED | string | Subject (the grant requester). |
| `aud` | REQUIRED | string | Audience (the grant target). |
| `iat` | REQUIRED | number | Issued-at timestamp (Unix seconds). |
| `exp` | REQUIRED | number | Expiration timestamp (Unix seconds). |
| `jti` | REQUIRED | string | JWT ID (UUID v4). |
| `grant_id` | REQUIRED | string | Reference to the grant. |
| `grant_type` | REQUIRED | string | The grant type (`once`, `timed`, `always`). |
| `approval` | OPTIONAL | string | The approval type (mirrors `grant_type`). |
| `permissions` | OPTIONAL | string[] | Granted permissions array. |
| `cmd_hash` | OPTIONAL | string | Command hash for verification. |
| `command` | OPTIONAL | string[] | Plaintext command array. |
| `decided_by` | OPTIONAL | string | Who approved the grant. |
| `nonce` | OPTIONAL | string | Request-specific nonce. |
| `run_as` | OPTIONAL | string | Execute command as this user identity. |

### 6.2 Expiration Rules

The `exp` claim is derived from the grant type:

| Grant Type | Expiration |
|------------|------------|
| `once` | `iat + 300` (5 minutes) |
| `timed` | `grant.expires_at` |
| `always` | `iat + 3600` (1 hour, renewable) |

### 6.3 Signing

The AuthZ-JWT MUST be signed with the IdP's signing key (the same key used for assertion JWTs, published in the IdP's JWKS).

### 6.4 Consumption Semantics

| Grant Type | On Consume |
|------------|------------|
| `once` | Grant transitions to `used`. The JWT becomes invalid for future consume calls. |
| `timed` | Grant remains `approved`. The JWT can be consumed multiple times until `expires_at`. |
| `always` | Grant remains `approved`. The JWT can be consumed indefinitely. A new JWT can be issued when the current one expires. |

---

## 7. Polling

Since the Grants Protocol uses polling (not callbacks) for status updates, clients SHOULD follow these guidelines:

### 7.1 Polling Interval

Clients SHOULD poll `GET /{id}` at intervals of **3–5 seconds**.

### 7.2 Conditional Requests

The server SHOULD support `ETag` / `If-None-Match` headers on `GET /{id}`:

- **Response** includes `ETag: "<value>"` header.
- **Subsequent requests** include `If-None-Match: "<value>"` header.
- If the grant has not changed, the server returns `304 Not Modified` (no body).

### 7.3 Retry-After

If the server is overloaded, it MAY return a `Retry-After` header with the polling response. Clients MUST respect this header and delay their next request accordingly.

### 7.4 Future Extension: Callbacks

A future extension MAY define callback/webhook URLs that the IdP notifies on status changes. This is out of scope for the current version.

---

## 8. Authentication & Authorization

### 8.1 Bearer Token

All API requests MUST include a valid Bearer token in the `Authorization` header:

```
Authorization: Bearer <token>
```

The token can be:
- A **session token** (for human users authenticated via WebAuthn).
- An **agent token** (for automated agents authenticated via Ed25519).

### 8.2 Permission Model

| Role | Can Create | Can List | Can Approve/Deny | Can Revoke | Can Get Token |
|------|-----------|---------|-----------------|-----------|--------------|
| **Requester** | Own grants | Own grants | No | No | Own approved grants |
| **Approver** | No | Grants targeting them or their agents | Yes | Yes | No |
| **Admin** | Any | All | Yes | Yes | No |

### 8.3 Agent Token Auto-Population

When a request is authenticated with an agent token, the `requester` field in grant creation MUST be automatically set to the authenticated agent's identity (the `sub` claim from the token). This prevents agents from creating grants on behalf of other entities.

---

## 9. Error Responses

### 9.1 Format

Grants Protocol errors follow [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) (Problem Details for HTTP APIs).

```json
{
  "type": "https://openape.org/errors/<error-type>",
  "title": "Human-readable title",
  "status": 400,
  "detail": "Detailed error description"
}
```

> **Note:** Grant errors use the `https://openape.org/errors/` URI base (not `https://ddisa.org/errors/`), since the Grants Protocol is part of the OpenAPE ecosystem, not DDISA Core.

### 9.2 Error Types

| Type | Status | Title | Description |
|------|--------|-------|-------------|
| `grant_not_found` | 404 | Grant Not Found | The specified grant ID does not exist. |
| `grant_already_decided` | 409 | Grant Already Decided | The grant has already been approved or denied. |
| `grant_expired` | 410 | Grant Expired | The timed grant has passed its expiration. |
| `grant_not_approved` | 400 | Grant Not Approved | The grant is not in `approved` status (required for token/consume). |
| `grant_already_used` | 410 | Grant Already Used | The `once` grant has already been consumed. |
| `invalid_grant_type` | 400 | Invalid Grant Type | The `grant_type` value is not supported. |
| `missing_duration` | 400 | Missing Duration | A `timed` grant was created without a `duration` field. |
| `batch_partial_failure` | 207 | Batch Partial Failure | Some operations in the batch succeeded while others failed. |
| `invalid_authz_jwt` | 401 | Invalid Authorization JWT | The AuthZ-JWT is malformed, expired, or has an invalid signature. |
| `unauthorized` | 401 | Unauthorized | Missing or invalid authentication token. |
| `forbidden` | 403 | Forbidden | Authenticated user lacks permission for this operation. |

---

## References

### Normative References

- [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) — Key words for use in RFCs
- [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) — Problem Details for HTTP APIs
- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) — Discovery mechanism

### Informative References

- [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) — OAuth 2.0 Token Revocation (conceptual model for revoke)
- [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) — OAuth 2.0 Token Introspection (conceptual model for introspect)
- [RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396) — OAuth 2.0 Rich Authorization Requests
- [DDISA Core Specification](core.md) — Core DNS discovery and authentication protocol
- [OpenAPE Delegation Protocol](delegation.md) — Delegation extension built on grants
