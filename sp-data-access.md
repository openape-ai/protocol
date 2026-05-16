# OpenApe SP Data Access Profile

**Version:** 1.0-draft
**Status:** Draft
**Date:** 2026-05-16
**Author:** Patrick Hofmann (Delta Mind GmbH)

## Abstract

This profile defines how one Service Provider (the **Receiver**, "SP2") obtains
a user's data from another Service Provider (the **Provider**, "SP1") with the
user's explicit, revocable consent — without any client registration. It is a
thin profile over the [Grants Protocol](grants.md) and the
[Delegation Protocol](delegation.md): the Receiver is a *delegate*, the consent
is a *delegation grant*. This document adds only what those do not cover:
**(a)** machine-discoverable provider scopes, **(b)** the DDISA trust doctrine
for issuer resolution, **(c)** provider-side scope enforcement, and
**(d)** consume semantics for reusable vs. one-shot data grants.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Trust Doctrine](#2-trust-doctrine)
3. [SP Scope Catalog (Discovery)](#3-sp-scope-catalog-discovery)
4. [Delegated Access Flow](#4-delegated-access-flow)
5. [Provider Conformance](#5-provider-conformance)
6. [Consume Semantics](#6-consume-semantics)
7. [Security Considerations](#7-security-considerations)
8. [Schemas](#8-schemas)

---

## 1. Introduction

### 1.1 Purpose

SP2 (Receiver) wants to read or write a user's data held by SP1 (Provider) —
e.g. an invoicing service pulling timesheets from `timetrack.openape.ai`. The
user authorizes this at **their own** IdP (discovered via their domain's DDISA
record). No party is pre-registered anywhere.

### 1.2 Requires

- [Grants Protocol](grants.md) — grant lifecycle, AuthZ-JWT, `/consume`.
- [Delegation Protocol](delegation.md) — a delegation is a grant with
  `type: "delegation"`, fields delegator/delegate/audience/scopes. **The
  Receiver is the delegate; its DDISA domain is the `act` actor.**
- [DDISA Core](core.md) §2–§4 — DNS discovery, IdP/SP metadata, JWKS.

This profile introduces **no new grant type and no new endpoint**. It
constrains and discovers existing ones.

## 2. Trust Doctrine

DDISA's premise is that **the DNS owner is the authority**. The `_ddisa`
record of a domain *is* the authority for identities in that domain. Not
trusting the record means not trusting the protocol.

Therefore:

> **2.1 (NORMATIVE)** For access to data owned by the subject, a Provider MUST
> resolve the trusted issuer **dynamically from the subject's domain** via
> DDISA (`_ddisa.<subject-domain>` → IdP → `jwks_uri`). A Provider MUST NOT
> hardcode the issuer and MUST NOT apply an issuer allowlist on this path. A
> domain owner running an IdP that grants whatever it deems correct **for its
> own subjects** is the intended behavior, not a threat.

> **2.2 (NORMATIVE)** An issuer **allowlist is permitted only** where the
> protected resource is owned by a party **other than** the token subject
> (subject ≠ resource-owner) — e.g. privilege elevation on a machine
> (`escapes`), or an SP guarding resources belonging to a third party. This is
> the resource-owner exercising *their own* DDISA-rooted policy and MUST be
> documented as an explicit exception, never the default.

> **2.3** Defense against a misbehaving issuer is **not** issuer distrust; it
> is authorization. A rogue IdP can only mint tokens for subjects of its own
> domain; the Provider's RBAC/scope layer ensures such a subject receives only
> what is theirs or what the actual data-owner granted (see §5.3).

## 3. SP Scope Catalog (Discovery)

### 3.1 Location

A Provider that supports this profile MUST expose a **scope catalog**. It
extends [core.md §4 SP Metadata](core.md#4-sp-metadata) and is published at:

```
{sp_url}/.well-known/openape.json   →  field: scopes
```

(The `/.well-known/oauth-client-metadata` document of core.md §4 MAY also link
it; `openape.json` is the OpenApe service manifest surface.)

### 3.2 Format

`scopes` is an array. Each entry:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Scope identifier. Convention: `<sp-shortname>:<action>`, e.g. `timetrack:read`, `timetrack:write`, `timetrack:report`. |
| `description` | string | Human-readable; rendered verbatim on the consent screen. |
| `grants` | string[] | Informative: API operations this scope authorizes, e.g. `"GET /api/me/entries"`. |

A Receiver MUST request only scope `id`s present in the catalog. A Provider
MUST reject an exchange whose requested scopes are not a subset of its catalog
(`invalid_scope`).

See [schemas/sp-scope-catalog.json](schemas/sp-scope-catalog.json).

## 4. Delegated Access Flow

This profile reuses the **existing** Delegation + Grants endpoints
([delegation.md](delegation.md), [grants.md](grants.md)) verbatim. It adds no
new endpoint.

```
SP2 (Receiver)                User-IdP                 SP1 (Provider)
  │  GET /.well-known/openape.json (SP1) ───────────────────►│  scopes
  │  resolve _ddisa.<user-domain> → User-IdP                 │
  │  POST {idp}/api/delegations                               │
  │    { delegate:<sp2-domain>, audience:<sp1>,               │
  │      scopes:⊆catalog }  (only act:'human' may create) ───►│
  │                          user consents (any channel)      │
  │  ◄── AuthZ-JWT via {idp}/api/grants/{id}/token            │
  │       (iss=user-idp, aud=<sp1>, scopes, delegate, jti) ───│
  │  POST {sp1}/api/cli/exchange  subject_token=<that JWT> ──►│ verify (§5)
  │  ◄────────────── SP1 access token (scope) ───────────────│
  │  GET {sp1}/api/...  Authorization: Bearer ───────────────►│ enforce (§5)
```

The delegate identity travels in the grant's `request.delegate` and the
issued AuthZ-JWT per [delegation.md](delegation.md) — this profile does **not**
introduce a separate `act` claim. The Provider treats `request.delegate` as
the actor (a DDISA domain string; no registration — the IdP accepts any
`delegate`, trust derives from the delegator's consent).

Issuance channel (browser consent screen, CLI `apes grant new`, or a
**standing grant** that auto-approves matching delegation requests) is a
Grants/Delegation concern — see [grants.md](grants.md) and
[delegation.md](delegation.md). All channels yield the same grant object.

## 5. Provider Conformance

A conforming Provider, on `POST /api/cli/exchange`:

- **5.1** MUST verify the `subject_token` signature against the JWKS resolved
  per §2.1 (subject-domain DDISA), and MUST verify `aud` equals the Provider's
  own origin/identifier. Tokens without `aud` MUST be rejected.
- **5.2** MUST verify requested `scope ⊆` published catalog (§3.2). The
  Receiver MAY further narrow scope at exchange; it MUST NOT widen it beyond
  the delegation grant.
- **5.3** MUST carry `scope` (and the delegate identity from
  `request.delegate`, per [delegation.md](delegation.md), as provenance) into
  the minted SP access token. Resource handlers MUST enforce scope (e.g. a
  `*:read` token MUST be rejected on a mutating route, `403`) **and** existing
  per-resource RBAC. For data the subject does not solely own (shared/other-
  user records), the data-owner's authority via their own DDISA + the RBAC
  matrix governs visibility — the subject's issuer does not. When the
  `subject_token` carries **no** `scope` (first-party `apes login`, not a
  delegation), the minted token is unrestricted exactly as before
  (behaviour-preserving).
- **5.4** MUST NOT issue long-lived access tokens for delegated access (see
  §6). The 30-day CLI-token TTL used for first-party CLI sessions does not
  apply here.

## 6. Consume Semantics

There is **no new `consume` field**. The two consume modes map onto existing
Grants Protocol mechanisms:

| Mode | Existing mechanism | Revocation latency | IdP coupling on data path |
|------|--------------------|--------------------|---------------------------|
| **reusable** (default for data access) | A **standing grant** (`grants.md`, `type:'standing'`, `status:'approved'`) that auto-approves matching delegation requests. The Provider mints a short-TTL access token (RECOMMENDED ≤ 15 min, no auto-refresh without a still-valid grant); the Receiver re-exchanges on expiry. A revoked/deleted standing grant fails the next exchange. | ≤ token TTL | none (loosely coupled — RECOMMENDED) |
| **one_shot** (high-stakes, the `escapes`/elevation pattern) | A normal delegation grant consumed via `POST {idp}/api/grants/{id}/consume` (`grants.md`); `used_at` makes it single-use, replay-proof. The Provider calls `/consume` on every use. | immediate | per use (justified for high-stakes) |

A Provider MAY require the one_shot path (mandatory `/consume`) for specific
high-risk scopes; otherwise the reusable path (standing grant + short-TTL
token) is the default. Revocation in both cases is the user revoking the grant
at their IdP (`POST /api/grants/{id}/revoke` or `DELETE /api/delegations/{id}`)
— no Provider-side allowlist or token blacklist is introduced.

## 7. Security Considerations

- **Issuer pinning is an anti-pattern here** (§2.1). Pinning belongs only to
  the subject≠resource-owner case (§2.2).
- **Audience confusion:** strict `aud` equality; missing `aud` rejected.
- **Downscoping only:** Provider rejects `scope ⊄ catalog`; Receiver may
  narrow, never widen past the grant.
- **Actor integrity:** the Receiver cannot set `act`; only the IdP sets it,
  after DDISA-verifying the delegate domain at grant time.
- **Replay:** `jti` + short TTL (`reusable`) or mandatory `/consume`
  (`one_shot`).
- **Standing grants** (reusable, no expiry) remove the human from the loop per
  request; they MUST be explicit opt-in, scope/actor-bound, prominently
  revocable, and every exchange MUST be audit-logged (`jti`, actor, scope, ts).

## 8. Schemas

| Schema | Reference |
|--------|-----------|
| [sp-scope-catalog.json](schemas/sp-scope-catalog.json) | §3.2 |
| [delegation.json](schemas/delegation.json) | delegation.md (the grant object used here) |
| [authz-jwt-claims.json](schemas/authz-jwt-claims.json) | grants.md (the subject_token claims) |
