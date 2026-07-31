# Changelog

All notable changes to the DDISA Protocol Specification will be documented in this file.

## [Unreleased]

### Added

- **SSH-key authentication** (`core.md` Section 5.4) — the `"ssh-key"` method that Section 3.2 already listed as a valid `ddisa_auth_methods_supported` value but never defined. A `private_key_jwt` client assertion, signed with the entity's registered Ed25519 SSH key, presented at the token endpoint. Includes the assertion claim set, the token request/response shape, and the OAuth error codes the token endpoint returns.
- **Programmatic delegation** (`delegation.md` Section 6.4) — how a delegate mints a delegated assertion without a browser, by naming `delegation_grant` and `audience` in the SSH-key flow. Section 6.1 previously said only "the delegate authenticates" and left the mechanism unspecified, so delegation had no programmatic path at all.
- `scope` added to the delegation assertion claims (`delegation.md` Section 5.1), mirroring the grant's scopes and REQUIRED (possibly `[]`) for delegated assertions.

### Changed

- `core.md` Section 5.1 now describes three authentication methods instead of two.
- Renumbered `core.md` Sections 5.4–5.6 to 5.5–5.7 to make room for the SSH-key flow. Cross-references updated in-repo; external links to those anchors need adjusting.

## [1.0-draft] - 2026-03-12

### Added

- **DDISA Core Specification** (`core.md`) — DNS discovery, IdP/SP metadata, WebAuthn and Ed25519 authentication, assertion JWT format, RFC 7807 error responses, security considerations.
- **OpenApe Grants Protocol** (`grants.md`) — Grant lifecycle, REST API (10 endpoints), AuthZ-JWT, cursor-based pagination, polling with ETag, batch operations, RFC 7807 error types.
- **OpenApe Delegation Protocol** (`delegation.md`) — User-to-user delegation, auto-approved delegations, RFC 8693 `act` claim, delegation validation, scope restrictions.
- **JSON Schemas** (`schemas/`) — Draft 2020-12 schemas for all data formats.
- **Example Flows** (`examples/`) — Complete HTTP request/response examples for all protocol flows.
- **README** with compliance levels (Core, Grants, Full) and reference implementation link.
