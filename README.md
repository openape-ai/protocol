# DDISA Protocol Specification

**DNS-Discoverable Identity & Service Authorization**

DDISA is an open protocol for decentralized identity on the web. It uses DNS TXT records to map email domains to Identity Providers, enabling any domain to control its own authentication infrastructure. The protocol builds on OAuth 2.0 and OpenID Connect, adding DNS-based discovery, WebAuthn passkey authentication, and Ed25519 challenge-response for programmatic agents.

## Documents

| Document | Description |
|----------|-------------|
| [core.md](core.md) | **DDISA Core** — DNS discovery, IdP/SP metadata, authentication flows (WebAuthn + Ed25519), token format, error responses |
| [grants.md](grants.md) | **Grants Protocol** — Grant-based authorization REST API, AuthZ-JWT, polling, pagination. Works with any OIDC IdP. |
| [delegation.md](delegation.md) | **Delegation Protocol** — User-to-user delegation of rights at Service Providers. Builds on Grants. |

## Schemas

Machine-readable [JSON Schema](https://json-schema.org/) (Draft 2020-12) definitions for all data formats:

| Schema | Spec Reference |
|--------|---------------|
| [ddisa-record.json](schemas/ddisa-record.json) | core.md Section 2 |
| [client-metadata.json](schemas/client-metadata.json) | core.md Section 4 |
| [openid-configuration-extensions.json](schemas/openid-configuration-extensions.json) | core.md Section 3 |
| [grant.json](schemas/grant.json) | grants.md Section 3 |
| [grant-request.json](schemas/grant-request.json) | grants.md Section 3.4 |
| [authz-jwt-claims.json](schemas/authz-jwt-claims.json) | grants.md Section 6 |
| [delegation.json](schemas/delegation.json) | delegation.md Section 3 |
| [error.json](schemas/error.json) | core.md Section 6 |

## Examples

Complete HTTP request/response examples for all protocol flows:

| Example | Description |
|---------|-------------|
| [discovery-flow.md](examples/discovery-flow.md) | DNS query, OIDC discovery, JWKS fetch, SP metadata |
| [auth-passkey-flow.md](examples/auth-passkey-flow.md) | Full OAuth 2.0 + PKCE + WebAuthn flow |
| [auth-ed25519-flow.md](examples/auth-ed25519-flow.md) | Ed25519 challenge-response for agents |
| [grant-lifecycle.md](examples/grant-lifecycle.md) | Create, poll, approve, token, consume |
| [delegation-flow.md](examples/delegation-flow.md) | Create delegation, act on behalf, validate, revoke |
| [error-examples.md](examples/error-examples.md) | RFC 7807 error responses for all error types |

## Compliance Levels

Implementations can claim compliance at three levels:

| Level | Requirements |
|-------|-------------|
| **DDISA Core** | Implements [core.md](core.md) — DNS discovery, OIDC metadata with `ddisa_*` extensions, at least one auth method (WebAuthn or Ed25519), assertion JWT format, RFC 7807 errors. |
| **DDISA Grants** | Implements Core + [grants.md](grants.md) — Grants REST API, AuthZ-JWT, polling. Advertises `openape_grants_endpoint` in OIDC discovery. |
| **DDISA Full** | Implements Core + Grants + [delegation.md](delegation.md) — Delegation API, RFC 8693 `act` claim, delegation validation. Advertises `openape_delegations_endpoint` in OIDC discovery. |

## Design Principles

- **DNS-first discovery** — Email address is the only input needed. No central registry.
- **OIDC-compatible** — Extends standard OAuth 2.0 / OIDC. Works with existing libraries.
- **Two equal auth methods** — WebAuthn for humans, Ed25519 for agents. No protocol-level distinction.
- **Grants are OIDC-independent** — The Grants Protocol works with any OIDC IdP, not just DDISA.
- **Polling over callbacks** — Edge-compatible, serverless-friendly. Callbacks as a future extension.
- **RFC 7807 errors** — Standard problem details format with typed error URIs.

## Reference Implementation

The reference implementation is [OpenAPE](https://github.com/openape-ai/openape), an open-source monorepo containing:

- `@openape/core` — DNS resolution, types, constants, JWT utilities
- `@openape/auth` — SP and IdP authentication logic
- `@openape/grants` — Grant lifecycle, AuthZ-JWT, delegation
- `@openape/nuxt-auth-idp` — Nuxt module for Identity Providers
- `@openape/nuxt-auth-sp` — Nuxt module for Service Providers
- `@openape/grapes` — CLI tool for grant management

## Future Extensions

- **Callback/Webhook Notifications** — IdP notifies requesters on grant status changes instead of polling.
- **Compliance Test Suite** — Automated tests for verifying spec compliance.
- **Algorithm Migration** — Mandating EdDSA (Ed25519) as the primary signing algorithm.

## Contributing

This specification is open for feedback and contributions. Please open an issue or pull request.

## License

[MIT](LICENSE) — This specification is released under the MIT License to encourage maximum adoption.
