# Changelog

All notable changes to the DDISA Protocol Specification will be documented in this file.

## [1.0-draft] - 2026-03-12

### Added

- **DDISA Core Specification** (`core.md`) — DNS discovery, IdP/SP metadata, WebAuthn and Ed25519 authentication, assertion JWT format, RFC 7807 error responses, security considerations.
- **OpenAPE Grants Protocol** (`grants.md`) — Grant lifecycle, REST API (10 endpoints), AuthZ-JWT, cursor-based pagination, polling with ETag, batch operations, RFC 7807 error types.
- **OpenAPE Delegation Protocol** (`delegation.md`) — User-to-user delegation, auto-approved delegations, RFC 8693 `act` claim, delegation validation, scope restrictions.
- **JSON Schemas** (`schemas/`) — Draft 2020-12 schemas for all data formats.
- **Example Flows** (`examples/`) — Complete HTTP request/response examples for all protocol flows.
- **README** with compliance levels (Core, Grants, Full) and reference implementation link.
