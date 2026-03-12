# DDISA Core Specification

**Version:** 1.0-draft
**Status:** Draft
**Date:** 2026-03-12
**Author:** Patrick Hofmann (Delta Mind GmbH)

## Abstract

DDISA (DNS-Discoverable Identity & Service Authorization) is a decentralized identity protocol that uses DNS TXT records to map email domains to Identity Providers (IdPs). Service Providers (SPs) discover a user's IdP through DNS, then authenticate the user via standard OAuth 2.0 / OpenID Connect flows extended with DDISA-specific metadata. This document specifies the core protocol: DNS discovery, IdP and SP metadata, authentication flows, token format, and error handling.

## Table of Contents

1. [Introduction](#1-introduction)
2. [DNS Discovery](#2-dns-discovery)
3. [IdP Metadata Discovery](#3-idp-metadata-discovery)
4. [SP Metadata](#4-sp-metadata)
5. [Authentication](#5-authentication)
6. [Error Responses](#6-error-responses)
7. [Security Considerations](#7-security-considerations)
8. [IANA Considerations](#8-iana-considerations)

---

## 1. Introduction

### 1.1 Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### 1.2 Roles

| Role | Description |
|------|-------------|
| **Identity Provider (IdP)** | Authenticates users and issues assertions. Hosts OIDC-compatible endpoints extended with DDISA metadata. |
| **Service Provider (SP)** | Relies on the IdP for user authentication. Publishes client metadata and validates assertions. |
| **User** | An entity (human or automated agent) that authenticates at an IdP to access an SP. DDISA does not distinguish between user types at the protocol level. |

### 1.3 Protocol Overview

```
  User                    SP                     DNS                    IdP
   |                       |                      |                      |
   |  1. Login with email  |                      |                      |
   |---------------------->|                      |                      |
   |                       |  2. Extract domain    |                      |
   |                       |  3. Query _ddisa.TXT  |                      |
   |                       |--------------------->|                      |
   |                       |  4. TXT record        |                      |
   |                       |<---------------------|                      |
   |                       |                      |                      |
   |                       |  5. Fetch /.well-known/openid-configuration |
   |                       |---------------------------------------------->|
   |                       |  6. IdP metadata                            |
   |                       |<----------------------------------------------|
   |                       |                      |                      |
   |  7. Redirect to IdP   |                      |                      |
   |<----------------------|                      |                      |
   |                       |                      |                      |
   |  8. Authenticate at IdP                      |                      |
   |------------------------------------------------------------->|
   |  9. Redirect with code                       |                      |
   |<-------------------------------------------------------------|
   |                       |                      |                      |
   |  10. Code to SP       |                      |                      |
   |---------------------->|                      |                      |
   |                       |  11. Exchange code for assertion            |
   |                       |---------------------------------------------->|
   |                       |  12. Assertion JWT                          |
   |                       |<----------------------------------------------|
   |                       |                      |                      |
   |                       |  13. Verify assertion via JWKS              |
   |                       |---------------------------------------------->|
   |                       |  14. JWKS response                          |
   |                       |<----------------------------------------------|
   |                       |                      |                      |
   |  15. Authenticated    |                      |                      |
   |<----------------------|                      |                      |
```

---

## 2. DNS Discovery

### 2.1 TXT Record Format

DDISA uses DNS TXT records to advertise which IdP is authoritative for a domain. The record MUST be placed at the subdomain `_ddisa.{domain}`.

**Format:**

```
v=ddisa1 idp=<URL>; mode=<mode>; priority=<N>; policy_endpoint=<URL>
```

The version tag `v=ddisa1` MUST be the first space-delimited token. Remaining fields are semicolon-delimited `key=value` pairs. Whitespace around separators is ignored.

**Example:**

```
_ddisa.example.com.  300  IN  TXT  "v=ddisa1 idp=https://id.example.com; mode=open; priority=10"
```

### 2.2 Record Placement

The TXT record MUST be placed at `_ddisa.{domain}`, where `{domain}` is the domain part of the user's email address.

For the email `alice@example.com`, the DNS lookup target is `_ddisa.example.com`.

### 2.3 Field Definitions

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `v` | REQUIRED | string | Version tag. MUST be `ddisa1`. |
| `idp` | REQUIRED | URL | The IdP base URL. MUST be an HTTPS URL. |
| `mode` | OPTIONAL | enum | Policy mode. One of: `open`, `allowlist-admin`, `allowlist-user`, `deny`. If omitted, IdP-specific default applies. |
| `priority` | OPTIONAL | integer | Priority value (lower = higher priority), similar to MX record priority. Default: `10`. |
| `policy_endpoint` | OPTIONAL | URL | URL for the IdP's policy management endpoint. |

**Policy Modes:**

| Mode | Description |
|------|-------------|
| `open` | Any SP may authenticate users without prior approval. |
| `allowlist-admin` | Only admin-approved SPs may authenticate users. |
| `allowlist-user` | Users are prompted to approve new SPs on first use. |
| `deny` | No SP authentication is permitted via this IdP. |

### 2.4 Resolution Algorithm

To resolve the IdP for an email address, an implementation MUST follow these steps:

1. **Extract domain** — Split the email at `@` and take the domain part.
2. **Query DNS** — Resolve TXT records for `_ddisa.{domain}`.
3. **Filter** — Select records where the first space-delimited token is `v=ddisa1`.
4. **Parse** — For each matching record, parse semicolon-delimited key-value fields.
5. **Validate** — Discard records missing the REQUIRED `idp` field.
6. **Sort** — Sort remaining records by `priority` (ascending). Records without a `priority` field default to `10`.
7. **Select** — Use the record with the lowest priority value.

If multiple records share the same priority, the implementation MAY select any of them.

### 2.5 DNS-over-HTTPS Fallback

Implementations running in environments without direct DNS access (browsers, edge runtimes, serverless) MUST support DNS-over-HTTPS (DoH) as a fallback resolution mechanism.

**Recommended DoH providers:**

| Provider | Endpoint |
|----------|----------|
| Cloudflare | `https://cloudflare-dns.com/dns-query` |
| Google | `https://dns.google/resolve` |
| Quad9 | `https://dns.quad9.net:5053/dns-query` |

DoH requests MUST use the `application/dns-json` content type via the `Accept` header.

**Example DoH request:**

```
GET https://cloudflare-dns.com/dns-query?type=TXT&name=_ddisa.example.com
Accept: application/dns-json
```

The response contains an `Answer` array. Implementations MUST filter answers by DNS record type `16` (TXT) and strip enclosing quotes from the `data` field.

### 2.6 Caching

Implementations SHOULD cache resolved DDISA records. Caches SHOULD respect the DNS TTL from the response. If no TTL is available (e.g., via DoH JSON responses that may not include TTL), the default cache duration is **300 seconds** (5 minutes).

Cache entries MUST be keyed by the bare domain (not the `_ddisa.` prefixed domain).

### 2.7 Fallback Behavior

If DNS discovery returns no valid DDISA record:

- The SP MAY fall back to a pre-configured IdP URL. The fallback URL is an SP configuration parameter and is not part of the DNS record or the DDISA protocol.
- The SP SHOULD inform the user that a fallback IdP is being used.
- The SP MUST NOT silently switch between IdPs without user awareness.

---

## 3. IdP Metadata Discovery

### 3.1 OIDC Discovery

A DDISA IdP MUST publish an OpenID Connect Discovery document at:

```
{idp_url}/.well-known/openid-configuration
```

This document MUST include all standard OIDC Discovery fields required by the IdP's supported flows, plus the DDISA extensions defined below.

### 3.2 DDISA Extensions (`ddisa_*` Namespace)

The following custom fields extend the standard OIDC Discovery document for DDISA-specific discovery:

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `ddisa_version` | REQUIRED | string | DDISA protocol version. MUST be `"1.0"`. |
| `ddisa_auth_methods_supported` | REQUIRED | string[] | Supported authentication methods. Values: `"webauthn"`, `"ed25519"`. |
| `ddisa_client_metadata_uri` | OPTIONAL | string | URL pointing to the IdP's own client metadata document (per [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)). Typically `/.well-known/oauth-client-metadata`. |
| `ddisa_agent_challenge_endpoint` | OPTIONAL | string | Endpoint for Ed25519 challenge requests. REQUIRED if `"ed25519"` is in `ddisa_auth_methods_supported`. |
| `ddisa_agent_authenticate_endpoint` | OPTIONAL | string | Endpoint for Ed25519 authentication. REQUIRED if `"ed25519"` is in `ddisa_auth_methods_supported`. |

### 3.3 OpenAPE Extensions (`openape_*` Namespace)

The following fields are part of the OpenAPE ecosystem and are independent of DDISA. They signal support for the Grants and Delegation protocols (see [grants.md](grants.md) and [delegation.md](delegation.md)).

| Field | Status | Type | Description |
|-------|--------|------|-------------|
| `openape_grants_endpoint` | OPTIONAL | string | Base URL for the Grants REST API. Signals Grants Protocol support. |
| `openape_delegations_endpoint` | OPTIONAL | string | Base URL for the Delegations REST API. Signals Delegation Protocol support. |
| `openape_grant_types_supported` | OPTIONAL | string[] | Supported grant types. Values: `"once"`, `"timed"`, `"always"`. |
| `openape_grant_categories_supported` | OPTIONAL | string[] | Supported grant categories. Values: `"command"`, `"delegation"`. |

### 3.4 JWKS

A DDISA IdP MUST publish a JSON Web Key Set (JWKS) at:

```
{idp_url}/.well-known/jwks.json
```

The JWKS MUST include at least one signing key. The key MUST include the `alg`, `use`, and `kid` fields.

> **Note:** The current reference implementation uses `ES256` (ECDSA with P-256). Future versions of this specification may mandate `EdDSA` (Ed25519). Implementations SHOULD be prepared to support both algorithms.

### 3.5 Required vs Optional Fields

**Minimum DDISA-compliant IdP Discovery Document:**

```json
{
  "issuer": "https://id.example.com",
  "authorization_endpoint": "https://id.example.com/authorize",
  "token_endpoint": "https://id.example.com/token",
  "jwks_uri": "https://id.example.com/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "code_challenge_methods_supported": ["S256"],
  "ddisa_version": "1.0",
  "ddisa_auth_methods_supported": ["webauthn"]
}
```

**Full example with optional fields:**

```json
{
  "issuer": "https://id.example.com",
  "authorization_endpoint": "https://id.example.com/authorize",
  "token_endpoint": "https://id.example.com/token",
  "jwks_uri": "https://id.example.com/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "code_challenge_methods_supported": ["S256"],
  "ddisa_version": "1.0",
  "ddisa_auth_methods_supported": ["webauthn", "ed25519"],
  "ddisa_client_metadata_uri": "https://id.example.com/.well-known/oauth-client-metadata",
  "ddisa_agent_challenge_endpoint": "https://id.example.com/api/agent/challenge",
  "ddisa_agent_authenticate_endpoint": "https://id.example.com/api/agent/authenticate",
  "openape_grants_endpoint": "https://id.example.com/api/grants",
  "openape_delegations_endpoint": "https://id.example.com/api/delegations",
  "openape_grant_types_supported": ["once", "timed", "always"],
  "openape_grant_categories_supported": ["command", "delegation"]
}
```

---

## 4. SP Metadata

### 4.1 Location

An SP MUST publish its client metadata at:

```
{sp_url}/.well-known/oauth-client-metadata
```

This follows [draft-ietf-oauth-client-metadata-discovery](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-metadata-discovery/).

> **Migration note:** Earlier versions used `/.well-known/sp-manifest.json`. Implementations encountering this legacy path SHOULD treat it as equivalent but SHOULD migrate to the standard path.

### 4.2 Basis

SP metadata follows [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) (OAuth 2.0 Dynamic Client Registration Metadata) field names.

### 4.3 Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `client_id` | string | Client identifier. Typically the SP's domain name. MUST be OIDC-compatible. |
| `client_name` | string | Human-readable display name of the SP. |
| `redirect_uris` | string[] | Array of allowed redirect URIs for OAuth callbacks. |

### 4.4 Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `jwks_uri` | string | URL to the SP's JWKS (for token verification). |
| `client_uri` | string | URL to the SP's homepage or description page. |
| `logo_uri` | string | URL to the SP's logo image. |
| `contacts` | string[] | Array of contact email addresses. |
| `policy_uri` | string | URL to the SP's privacy policy. |
| `tos_uri` | string | URL to the SP's terms of service. |

### 4.5 DDISA Extensions

SP metadata MAY include DDISA-specific fields, prefixed with `ddisa_`:

| Field | Type | Description |
|-------|------|-------------|
| `ddisa_domain` | string | The SP's primary domain (for DDISA DNS verification). |

### 4.6 Migration from Legacy Format

| RFC 7591 Field | Legacy OpenAPE Field | Notes |
|----------------|---------------------|-------|
| `client_name` | `name` | Renamed to standard |
| `contacts` | `contact` | Changed from string to string[] |
| `client_uri` | `description` | Semantics differ slightly; `client_uri` is a URL |

### 4.7 Example

```json
{
  "client_id": "app.example.com",
  "client_name": "Example App",
  "redirect_uris": [
    "https://app.example.com/auth/callback"
  ],
  "jwks_uri": "https://app.example.com/.well-known/jwks.json",
  "client_uri": "https://app.example.com",
  "logo_uri": "https://app.example.com/logo.png",
  "contacts": ["admin@example.com"],
  "policy_uri": "https://app.example.com/privacy",
  "tos_uri": "https://app.example.com/terms"
}
```

---

## 5. Authentication

### 5.1 Overview

DDISA supports two equivalent authentication methods. Neither method is privileged over the other — both produce the same assertion JWT format. The choice of method depends on the authenticating entity's capabilities:

| Method | Use Case | Mechanism |
|--------|----------|-----------|
| **WebAuthn** | Browser-based authentication | OAuth 2.0 Authorization Code + PKCE |
| **Ed25519** | Programmatic / agent authentication | Challenge-response with Ed25519 signatures |

An IdP MUST support at least one method and MUST advertise supported methods via `ddisa_auth_methods_supported` in the discovery document.

### 5.2 WebAuthn Flow (OAuth 2.0 Authorization Code + PKCE)

This flow follows standard [OAuth 2.0 Authorization Code](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1) with [PKCE](https://datatracker.ietf.org/doc/html/rfc7636).

#### 5.2.1 Authorization Request

The SP redirects the user to the IdP's `authorization_endpoint` with the following parameters:

| Parameter | Status | Description |
|-----------|--------|-------------|
| `response_type` | REQUIRED | MUST be `"code"`. |
| `client_id` | REQUIRED | The SP's client identifier. |
| `redirect_uri` | REQUIRED | Callback URL. MUST be registered in the SP's metadata. |
| `state` | REQUIRED | Opaque CSRF protection value. |
| `code_challenge` | REQUIRED | PKCE code challenge (Base64url-encoded SHA-256 hash of the code verifier). |
| `code_challenge_method` | REQUIRED | MUST be `"S256"`. Plain challenges are NOT supported. |
| `nonce` | REQUIRED | Replay protection nonce. MUST be included in the resulting assertion. |
| `login_hint` | OPTIONAL | The user's email address. Allows the IdP to pre-fill the login form. |

**Example:**

```
GET /authorize?response_type=code
  &client_id=app.example.com
  &redirect_uri=https%3A%2F%2Fapp.example.com%2Fauth%2Fcallback
  &state=abc123
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  &nonce=xyz789
```

#### 5.2.2 Authorization Response

After successful authentication, the IdP redirects back to the SP's `redirect_uri`:

```
HTTP/1.1 302 Found
Location: https://app.example.com/auth/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=abc123
```

#### 5.2.3 Token Exchange

The SP exchanges the authorization code for an assertion via backchannel POST to the IdP's `token_endpoint`:

**Request:**

```
POST /token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "code": "SplxlOBeZQQYbYS6WxSbIA",
  "code_verifier": "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk",
  "redirect_uri": "https://app.example.com/auth/callback",
  "client_id": "app.example.com"
}
```

**Response:**

```json
{
  "assertion": "<JWT>",
  "authorization_details": []
}
```

The `assertion` field contains the signed JWT (see [Section 5.4](#54-token-format)).

The `authorization_details` field is OPTIONAL and follows [RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396) for Rich Authorization Requests.

### 5.3 Ed25519 Challenge-Response Flow

This flow enables programmatic authentication using Ed25519 key pairs. It is designed for automated agents, CLI tools, and service-to-service communication.

#### 5.3.1 Prerequisites

The authenticating entity MUST have:
1. An Ed25519 key pair registered with the IdP.
2. An agent identifier (ID or email) known to the IdP.

#### 5.3.2 Challenge Request

**Request:**

```
POST {ddisa_agent_challenge_endpoint}
Content-Type: application/json

{
  "agent_id": "agent@example.com"
}
```

The `agent_id` field MUST be either the agent's unique ID or its registered email address.

**Response:**

```json
{
  "challenge": "random-challenge-string"
}
```

The challenge MUST be:
- Cryptographically random
- Single-use (consumed on verification)
- Time-limited (implementation-defined, RECOMMENDED: 5 minutes)

#### 5.3.3 Authentication Request

The agent signs the challenge with its Ed25519 private key and submits:

**Request:**

```
POST {ddisa_agent_authenticate_endpoint}
Content-Type: application/json

{
  "agent_id": "agent@example.com",
  "challenge": "random-challenge-string",
  "signature": "<base64-encoded-ed25519-signature>"
}
```

The `signature` MUST be the Ed25519 signature of the exact `challenge` string, encoded as Base64.

**Response:**

```json
{
  "token": "<JWT>",
  "agent_id": "agent-uuid",
  "email": "agent@example.com",
  "name": "My Agent",
  "expires_in": 3600
}
```

The `token` field contains a signed JWT. The `expires_in` field indicates the token's validity in seconds.

#### 5.3.4 Agent Enrollment

Agent enrollment (registering new Ed25519 key pairs) is an IdP-specific operation and is outside the scope of this specification. Implementations MAY provide enrollment endpoints, but the mechanism is not standardized.

### 5.4 Token Format

Both authentication methods produce a JWT (JSON Web Token) as the assertion. The JWT MUST be signed using a key published in the IdP's JWKS.

> **Note:** The current reference implementation uses `ES256`. Future versions may mandate `EdDSA`. Implementations SHOULD support both and MUST NOT negotiate algorithms — the algorithm is determined by the IdP's signing key.

#### 5.4.1 Assertion Claims

| Claim | Status | Type | Description |
|-------|--------|------|-------------|
| `iss` | REQUIRED | string | Issuer. MUST match the IdP URL discovered via DNS. |
| `sub` | REQUIRED | string | Subject identifier (user or agent email/ID). |
| `aud` | REQUIRED | string | Audience. MUST match the SP's `client_id`. |
| `iat` | REQUIRED | number | Issued-at timestamp (Unix seconds). |
| `exp` | REQUIRED | number | Expiration timestamp (Unix seconds). `exp - iat` MUST NOT exceed **300 seconds** (5 minutes). |
| `nonce` | CONDITIONAL | string | REQUIRED for the WebAuthn flow. MUST match the nonce from the authorization request. OPTIONAL for Ed25519 flow. |
| `jti` | OPTIONAL | string | JWT ID for deduplication. |
| `act` | OPTIONAL | string \| object | Actor claim. If a string: free-form actor type (e.g., `"human"`, `"agent"`). If an object: RFC 8693 delegation claim with `{ "sub": "<delegate>" }`. See [delegation.md](delegation.md). |
| `delegate` | OPTIONAL | object | Delegate information for delegation scenarios. See [delegation.md](delegation.md). |
| `delegation_grant` | OPTIONAL | string | Grant ID authorizing the delegation. See [delegation.md](delegation.md). |

#### 5.4.2 Example Assertion (Decoded Payload)

```json
{
  "iss": "https://id.example.com",
  "sub": "alice@example.com",
  "aud": "app.example.com",
  "iat": 1710244800,
  "exp": 1710245100,
  "nonce": "xyz789",
  "jti": "unique-jwt-id"
}
```

### 5.5 Assertion Verification

An SP receiving an assertion MUST perform the following validation steps:

1. **Signature** — Verify the JWT signature using the IdP's JWKS (fetched from `{idp_url}/.well-known/jwks.json`). The `kid` in the JWT header MUST match a key in the JWKS.
2. **Issuer** — The `iss` claim MUST match the IdP URL obtained during DNS discovery.
3. **Audience** — The `aud` claim MUST match the SP's own `client_id`.
4. **Expiration** — The `exp` claim MUST be in the future. The `exp - iat` difference MUST NOT exceed 300 seconds.
5. **Nonce** — If the SP sent a `nonce` in the authorization request, the `nonce` claim in the assertion MUST match.

If any validation step fails, the SP MUST reject the assertion.

### 5.6 Policy Evaluation

After successful authentication, the IdP evaluates the `mode` from the DNS record (if present) to determine whether the SP is allowed to receive an assertion:

| Mode | Behavior |
|------|----------|
| `open` | Issue assertion immediately. |
| `allowlist-user` | Check if the user has previously consented to this SP. If not, prompt for consent. |
| `allowlist-admin` | Check if the SP is on an admin-maintained allowlist. If not, deny. |
| `deny` | Reject the authorization request. |
| *(not set)* | IdP-specific default behavior (RECOMMENDED: prompt for consent). |

---

## 6. Error Responses

### 6.1 Format

DDISA error responses follow [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) (Problem Details for HTTP APIs).

```json
{
  "type": "https://ddisa.org/errors/<error-type>",
  "title": "Human-readable title",
  "status": 400,
  "detail": "Detailed error description"
}
```

The `Content-Type` for error responses MUST be `application/problem+json`.

### 6.2 Error Types

| Type | Status | Title | Description |
|------|--------|-------|-------------|
| `invalid_record` | 400 | Invalid DDISA Record | The DNS TXT record is malformed or missing required fields. |
| `idp_unreachable` | 502 | IdP Unreachable | The IdP specified in the DNS record could not be contacted. |
| `invalid_manifest` | 400 | Invalid Client Metadata | The SP's client metadata document is malformed or missing required fields. |
| `invalid_token` | 401 | Invalid Token | The assertion JWT is malformed, has an invalid signature, or has invalid claims. |
| `token_expired` | 401 | Token Expired | The assertion JWT has expired (`exp` is in the past). |
| `invalid_audience` | 401 | Invalid Audience | The `aud` claim does not match the expected `client_id`. |
| `invalid_nonce` | 401 | Invalid Nonce | The `nonce` claim does not match the expected nonce. |
| `unsupported_auth_method` | 400 | Unsupported Auth Method | The requested authentication method is not supported by this IdP. |
| `policy_denied` | 403 | Access Denied by Policy | The IdP's policy denies authentication for this SP. |
| `invalid_pkce` | 400 | Invalid PKCE | The PKCE code verifier does not match the code challenge. |
| `invalid_state` | 400 | Invalid State | The `state` parameter does not match (possible CSRF). |

### 6.3 HTTP Status Code Mapping

Error responses MUST use standard HTTP status codes:

- **400** — Client errors (malformed requests, invalid parameters)
- **401** — Authentication errors (invalid tokens, expired tokens)
- **403** — Authorization errors (policy denied)
- **404** — Resource not found
- **502** — Upstream errors (IdP unreachable)

---

## 7. Security Considerations

### 7.1 DNS Security

- DDISA records SHOULD be protected with [DNSSEC](https://datatracker.ietf.org/doc/html/rfc4033) to prevent DNS spoofing attacks.
- When using DNS-over-HTTPS, implementations MUST verify TLS certificates of the DoH provider.
- Implementations SHOULD support multiple DoH providers as fallbacks.

### 7.2 Transport Security

- All IdP and SP endpoints MUST be served over HTTPS (TLS 1.2 or later).
- The `idp` field in the DNS record MUST be an HTTPS URL.
- Redirect URIs MUST use HTTPS (with the exception of `http://localhost` for local development).

### 7.3 PKCE

- DDISA mandates PKCE with `S256` for all authorization code flows.
- The `plain` code challenge method MUST NOT be supported.
- Code verifiers MUST be cryptographically random with at least 256 bits of entropy.

### 7.4 Key Management

- IdPs MUST rotate signing keys periodically.
- The JWKS MUST include the `kid` field for key identification.
- SPs MUST support key rollover by checking the `kid` in the JWT header against the JWKS.
- Ed25519 private keys for agent authentication MUST be stored securely and MUST NOT be transmitted.

### 7.5 Token Security

- Assertion JWTs MUST have a maximum lifetime of 300 seconds (5 minutes).
- Assertions SHOULD include the `jti` claim for replay detection.
- SPs SHOULD maintain a short-lived cache of consumed `jti` values to prevent replay attacks.

### 7.6 Nonce Handling

- The WebAuthn flow MUST include a `nonce` in both the authorization request and the resulting assertion.
- SPs MUST validate the nonce before accepting the assertion.
- Nonces MUST be cryptographically random and single-use.

### 7.7 Challenge Security (Ed25519)

- Challenges MUST be cryptographically random.
- Challenges MUST be single-use (consumed after successful verification).
- Challenges SHOULD expire after a short period (RECOMMENDED: 5 minutes).
- The IdP MUST NOT reissue the same challenge value.

---

## 8. IANA Considerations

### 8.1 `ddisa_*` Namespace

This specification defines custom fields in the `ddisa_*` namespace for use in OIDC Discovery documents and SP metadata. These fields are not registered with IANA but follow the convention of vendor-prefixed extensions permitted by the OpenID Connect Discovery specification.

### 8.2 Well-Known URIs

This specification uses the following Well-Known URIs:

| URI | Specification |
|-----|---------------|
| `/.well-known/openid-configuration` | [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) |
| `/.well-known/jwks.json` | Common convention (referenced by `jwks_uri` in OIDC Discovery) |
| `/.well-known/oauth-client-metadata` | [draft-ietf-oauth-client-metadata-discovery](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-metadata-discovery/) |

### 8.3 DNS Record

DDISA uses the `_ddisa.` subdomain prefix for TXT records. This follows the convention established by [RFC 6763](https://datatracker.ietf.org/doc/html/rfc6763) for DNS-based service discovery using underscore-prefixed labels.

---

## References

### Normative References

- [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) — Key words for use in RFCs
- [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) — OAuth 2.0 Authorization Framework
- [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) — PKCE for OAuth 2.0
- [RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591) — OAuth 2.0 Dynamic Client Registration Metadata
- [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) — Problem Details for HTTP APIs
- [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) — OAuth 2.0 Token Exchange
- [RFC 9396](https://datatracker.ietf.org/doc/html/rfc9396) — OAuth 2.0 Rich Authorization Requests

### Informative References

- [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [WebAuthn Level 2](https://www.w3.org/TR/webauthn-2/)
- [RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033) — DNSSEC Introduction
- [RFC 6763](https://datatracker.ietf.org/doc/html/rfc6763) — DNS-Based Service Discovery
- [draft-ietf-oauth-client-metadata-discovery](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-metadata-discovery/) — OAuth Client Metadata Discovery
