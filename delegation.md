# OpenAPE Delegation Protocol

**Version:** 1.0-draft
**Status:** Draft
**Date:** 2026-03-12
**Author:** Patrick Hofmann (Delta Mind GmbH)

## Abstract

The OpenAPE Delegation Protocol enables users to delegate rights to other users or agents at specific Service Providers. Delegations are modeled as a specialized category of grants (see [grants.md](grants.md)) with additional fields for delegator, delegate, audience, and scopes. This protocol requires the OpenAPE Grants Protocol but is independent of DDISA Core.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Discovery](#2-discovery)
3. [Data Model](#3-data-model)
4. [REST API](#4-rest-api)
5. [Delegation JWT Claims](#5-delegation-jwt-claims)
6. [Flows](#6-flows)
7. [Security Considerations](#7-security-considerations)

---

## 1. Introduction

### 1.1 Purpose

The Delegation Protocol allows User A (the delegator) to authorize User B (the delegate) to act on their behalf at Service C (the audience). The delegation is scoped — it specifies which actions the delegate may perform and at which service.

### 1.2 Requires: OpenAPE Grants Protocol

This protocol builds on the [OpenAPE Grants Protocol](grants.md). A delegation is a grant with `type: "delegation"` and additional delegation-specific fields. All grant lifecycle operations (creation, approval, revocation, expiration) apply to delegations.

DDISA Core is NOT required — any OIDC IdP implementing the Grants Protocol can support delegations.

### 1.3 Delegation vs Grant

| Aspect | Command Grant | Delegation Grant |
|--------|--------------|-----------------|
| Purpose | Authorize a specific action | Authorize acting on behalf of someone |
| Category (`type`) | `"command"` (default) | `"delegation"` |
| Approval | Requires explicit approval | Auto-approved by delegator on creation |
| Key Fields | `requester`, `target`, `command` | `delegator`, `delegate`, `audience`, `scopes` |
| JWT Claims | Standard AuthZ-JWT | Includes `act` (RFC 8693) and `delegate` claims |

---

## 2. Discovery

An IdP that supports the Delegation Protocol MUST advertise its support via the OIDC Discovery document.

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `openape_delegations_endpoint` | REQUIRED | string | Base URL for the Delegations REST API. |
| `openape_delegation_scopes_supported` | OPTIONAL | string[] | List of supported delegation scopes. If omitted, any scope string is accepted. |

The IdP MUST also include `"delegation"` in `openape_grant_categories_supported`.

**Example:**

```json
{
  "openape_delegations_endpoint": "https://id.example.com/api/delegations",
  "openape_grant_categories_supported": ["command", "delegation"],
  "openape_delegation_scopes_supported": ["read", "write", "admin"]
}
```

---

## 3. Data Model

A delegation is a grant (see [grants.md Section 3.1](grants.md#31-grant-object)) with `type: "delegation"` and delegation-specific fields in the `request` object.

### 3.1 Delegation-Specific Request Fields

These fields are part of the grant `request` object and are REQUIRED for delegation grants:

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `delegator` | REQUIRED | string | The user granting the delegation (their email/ID). Automatically set to the authenticated user. |
| `delegate` | REQUIRED | string | The user or agent receiving the delegation (their email/ID). |
| `audience` | REQUIRED | string | The SP where the delegation is valid. Use `"*"` for any SP (NOT RECOMMENDED). |
| `scopes` | OPTIONAL | string[] | Actions the delegate may perform. If omitted, the delegation grants full access at the audience SP. |
| `grant_type` | REQUIRED | string | One of: `"once"`, `"timed"`, `"always"`. Controls delegation lifetime. |
| `duration` | CONDITIONAL | number | Duration in seconds. REQUIRED when `grant_type` is `"timed"`. |

### 3.2 Example Delegation Grant Object

```json
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

---

## 4. REST API

All endpoints are relative to `{openape_delegations_endpoint}` as advertised in the OIDC Discovery document.

### 4.1 Create Delegation

```
POST /
```

Creates a new delegation grant. The delegation is **auto-approved** by the delegator (the authenticated user).

**Request Body:**

```json
{
  "delegate": "bob@example.com",
  "audience": "app.example.com",
  "scopes": ["read", "write"],
  "grant_type": "timed",
  "duration": 86400
}
```

**Required Fields:** `delegate`, `audience`.

**Defaults:** `grant_type` defaults to `"once"` if omitted.

**Authentication:** The authenticated user becomes the `delegator`. Session authentication is REQUIRED (agent tokens MUST NOT create delegations).

**Side Effects:**

1. Creates a grant with `type: "delegation"` and `status: "pending"`.
2. Immediately auto-approves the grant (sets `status: "approved"`, `decided_by: <delegator>`).
3. For `timed` grants: sets `expires_at`.

**Response:** `201 Created` with the full Grant object (already in `approved` status).

### 4.2 List Delegations

```
GET /?role=<role>
```

Returns delegations visible to the authenticated user.

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `role` | string | Filter: `"delegator"` (created by me) or `"delegate"` (created for me). |

**Behavior:** Returns the union of delegations where the user is the delegator OR the delegate, deduplicated by ID, sorted by `created_at` descending.

**Response:** `200 OK` with an array of Grant objects where `type === "delegation"`.

### 4.3 Get Delegation

```
GET /{id}
```

Returns a single delegation by ID.

**Response:** `200 OK` with the Grant object. Returns `404` if not found or if the grant is not a delegation.

### 4.4 Revoke Delegation

```
DELETE /{id}
```

Revokes an active delegation.

**Preconditions:**

- The grant MUST be a delegation (`type: "delegation"`).
- The grant MUST be in `approved` status.
- The authenticated user MUST be the delegator.

**Response:** `200 OK` with the updated Grant object (status: `revoked`).

### 4.5 Validate Delegation

```
POST /{id}/validate
```

Validates whether a delegation is currently valid for a specific delegate and audience combination.

**Request Body:**

```json
{
  "delegate": "bob@example.com",
  "audience": "app.example.com"
}
```

**Validation Steps:**

1. Grant MUST exist and have `type: "delegation"`.
2. Grant MUST be in `approved` status (auto-expires timed grants).
3. `request.delegate` MUST match the provided `delegate`.
4. `request.audience` MUST match the provided `audience` (or be `"*"`).

**Response (valid):**

```json
{
  "valid": true,
  "delegation": { ... },
  "scopes": ["read", "write"]
}
```

**Response (invalid):**

```json
{
  "valid": false,
  "error": "Delegation grant is not approved: expired"
}
```

---

## 5. Delegation JWT Claims

When a delegate authenticates and acts on behalf of a delegator, the resulting assertion JWT includes delegation-specific claims.

### 5.1 Assertion Claims for Delegation

In addition to the standard assertion claims (see [core.md Section 5.4.1](core.md#541-assertion-claims)), delegation adds:

| Claim | Status | Type | Description |
|-------|--------|------|-------------|
| `sub` | REQUIRED | string | The **delegator's** identity (the person being acted on behalf of). |
| `act` | REQUIRED | object | RFC 8693 actor claim: `{ "sub": "<delegate-identity>" }`. |
| `delegate` | OPTIONAL | object | Extended delegate information (see below). |
| `delegation_grant` | OPTIONAL | string | The grant ID authorizing this delegation. |

### 5.2 `act` Claim (RFC 8693)

The `act` (actor) claim identifies who is actually performing the action:

```json
{
  "act": {
    "sub": "bob@example.com"
  }
}
```

When `act` is an object (not a string), the SP knows this is a delegated action. The `sub` at the top level is the delegator; `act.sub` is the delegate.

### 5.3 `delegate` Claim

The `delegate` claim provides additional context about the delegation:

```json
{
  "delegate": {
    "sub": "agent@example.com",
    "act": "agent",
    "grant_id": "d1e2f3a4-..."
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `sub` | string | The delegate's identity. |
| `act` | string | The delegate's actor type (e.g., `"agent"`). |
| `grant_id` | string | The delegation grant ID. |

### 5.4 Example Delegation Assertion

```json
{
  "iss": "https://id.example.com",
  "sub": "alice@example.com",
  "aud": "app.example.com",
  "iat": 1710244800,
  "exp": 1710245100,
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

---

## 6. Flows

### 6.1 User-to-User Delegation

```
  Alice (Delegator)        IdP                    Bob (Delegate)         SP
     |                      |                        |                    |
     |  1. POST /delegations|                        |                    |
     |  (delegate: bob,     |                        |                    |
     |   audience: sp.com)  |                        |                    |
     |--------------------->|                        |                    |
     |  2. 201 Created      |                        |                    |
     |  (auto-approved)     |                        |                    |
     |<---------------------|                        |                    |
     |                      |                        |                    |
     |                      |  3. Bob authenticates  |                    |
     |                      |<-----------------------|                    |
     |                      |  4. Assertion JWT       |                    |
     |                      |  (sub: alice,          |                    |
     |                      |   act: { sub: bob })   |                    |
     |                      |----------------------->|                    |
     |                      |                        |                    |
     |                      |                        |  5. Access SP      |
     |                      |                        |  (with delegation  |
     |                      |                        |   JWT)             |
     |                      |                        |------------------->|
     |                      |                        |  6. SP validates   |
     |                      |                        |<-------------------|
```

### 6.2 SP Validates Delegation

When an SP receives an assertion with delegation claims, it MUST:

1. Verify the assertion JWT signature and standard claims (see [core.md Section 5.5](core.md#55-assertion-verification)).
2. Check that `act` is an object (indicating delegation, not direct auth).
3. Optionally call `POST /{delegation_grant}/validate` at the IdP to verify the delegation is still active and the scopes are sufficient.
4. Apply scope restrictions — the delegate SHOULD only be allowed to perform actions within the delegation's `scopes`.

### 6.3 Delegation Revocation

```
  Alice (Delegator)        IdP
     |                      |
     |  DELETE /delegations/{id}
     |--------------------->|
     |  200 OK (revoked)    |
     |<---------------------|
```

After revocation, any subsequent validation requests for this delegation MUST return `valid: false`.

---

## 7. Security Considerations

### 7.1 Least Privilege

- Delegations SHOULD use the narrowest possible `scopes`.
- Delegations SHOULD use `"once"` or `"timed"` grant types when possible.
- `"always"` delegations SHOULD be avoided for sensitive operations.

### 7.2 Audience Restriction

- The `audience` field MUST restrict where the delegation is valid.
- Using `"*"` (any SP) is NOT RECOMMENDED and SHOULD be limited to trusted delegates.
- SPs MUST verify that their own `client_id` matches the delegation's `audience`.

### 7.3 No Chaining

Delegation chaining (delegate B further delegates to delegate C) is NOT supported. The maximum delegation depth is **1 level**:

```
Delegator → Delegate  ✓
Delegator → Delegate → Sub-delegate  ✗
```

Implementations MUST reject delegation creation attempts where the `delegator` is already acting as a delegate.

### 7.4 Delegator Authentication

Only session-authenticated users (not agents) MAY create delegations. This ensures that the delegator has directly authenticated and consciously authorized the delegation.

### 7.5 Revocation Propagation

SPs that cache delegation status SHOULD re-validate delegations periodically (RECOMMENDED: every 5 minutes) or on each use. Cached delegation validity MUST NOT exceed the delegation's `expires_at`.

---

## References

### Normative References

- [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) — Key words for use in RFCs
- [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) — OAuth 2.0 Token Exchange (act claim)
- [OpenAPE Grants Protocol](grants.md) — Grant lifecycle and REST API

### Informative References

- [DDISA Core Specification](core.md) — Core DNS discovery and authentication protocol
