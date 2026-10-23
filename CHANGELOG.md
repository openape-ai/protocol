# Changelog

## 2026-10-23

- Add terminal grant status `withdrawn` and requester withdrawal of its own referenced single-use request (`POST /withdraw`, discovery `openape_grant_withdraw_supported`), including approved but unconsumed grants. Add broker operation `withdraw` with signed reference and optional reason, scoped to the original binding. Withdrawn requests leave pending listings and limits and are never renewed.

## 2026-10-20

- Add optional generic grant references, atomic authority-scoped request deduplication, immutable binding conflicts and terminal retention. Define non-consuming reference lookup, signed broker get-by-reference, explicit support discovery and terminal execution refusal. References remain independent of cross-run batch grouping.

## 2026-10-18

- Align the broker assertion schema and specification with the existing `authorize` per-call operation and strict program request bounds. This documents existing implementation behavior rather than enabling a new authorization operation.
- Add negotiated verified broker-registered `requester_name` on signed non-connection broker assertions and stored Grant display metadata. The optional discovery flag `openape_broker_requester_name_supported` prevents sending new claims to older strict peers. Names are bounded, presentation-only and excluded from authorization/token provenance and binding equality; no requester-written or unsigned identity is trusted.

- Add optional requester-authored `batch.description` (at most 2000 Unicode code points, LF preserved, other Cc/Cf/Zl/Zp characters rejected). Members share an identical description, including whether it is present. The field is presentation-only; legacy batches without it and per-member authorization/decision semantics remain unchanged.

## 2026-10-14

- Replace the grant request `summary` member `text` with `title` (REQUIRED, one line, 1–120 characters), `description` (at most 2000 characters, line breaks kept) and `origin` (one line, at most 120 characters, set by the requester's runtime); `link` is now limited to 2048 characters. Unicode control, format and line/paragraph separator characters are refused in every member except LF in `description`, and an IdP rejects any other member including the former `text`. Existing grants are expected to be migrated (first non-empty line to `title`, the rest to `description`).
- Add delegations at the IdP's own grants API (delegation.md §8): the scopes `grants:read` and `grants:request` let a user's own agent read the user's grants and file pending requests for the user's agents, with the IdP issuer as audience. Deciding grants stays non-delegable; every other IdP endpoint refuses delegated tokens. Grant requests gain the server-set `submitted_by` provenance.

## 2026-10-07

- Add the optional display-only `request.batch` grouping (`id`, `title`, `size`) to grant requests, the `batch` list filter and the `openape_grant_batch_supported` discovery flag. Batches never change the authorization semantics of their members; members must be uniform `once` grants of one audience and target host. Batch operations are authorized per operation like the single-grant endpoints.

## 2026-09-19

- Add the optional Grant Brokering Profile: external agent identities, owner-scoped broker consent, signed mediated requests, original decision-provider signatures and revocation-aware execution. Extend discovery and grant schemas with profile metadata and immutable provenance.

All notable changes to the DDISA Protocol Specification will be documented in this file.

## [1.0-draft] - 2026-03-12

### Added

- **DDISA Core Specification** (`core.md`) — DNS discovery, IdP/SP metadata, WebAuthn and Ed25519 authentication, assertion JWT format, RFC 7807 error responses, security considerations.
- **OpenApe Grants Protocol** (`grants.md`) — Grant lifecycle, REST API (10 endpoints), AuthZ-JWT, cursor-based pagination, polling with ETag, batch operations, RFC 7807 error types.
- **OpenApe Delegation Protocol** (`delegation.md`) — User-to-user delegation, auto-approved delegations, RFC 8693 `act` claim, delegation validation, scope restrictions.
- **JSON Schemas** (`schemas/`) — Draft 2020-12 schemas for all data formats.
- **Example Flows** (`examples/`) — Complete HTTP request/response examples for all protocol flows.
- **README** with compliance levels (Core, Grants, Full) and reference implementation link.
