# OpenApe Grant Brokering Profile

**Version:** 1.0-draft  
**Status:** Draft  
**Date:** 2026-09-19

## 1. Purpose and roles

An agent can authenticate at one DDISA Identity Provider while its human owner decides command grants at another. The agent's provider mediates requests; the owner's provider retains the grant record, decision, signing key and consumption state. No local agent account is required at the decision provider.

This profile extends [Grants](grants.md), [DDISA Core](core.md) and their schemas. It does not delegate permission to approve or sign execution grants. It is not the chained delegation prohibited by [Delegation §7.3](delegation.md#73-no-chaining).

- **Agent provider / broker:** authenticates agents, maintains their owner bindings and forwards requests. In this version these are the same issuer.
- **Decision provider:** authenticates the human, records their explicit broker consent, decides and signs grants.
- **Owner:** a directly authenticated human at the decision provider.
- **Executor:** enforces an owner-authorized grant on a particular machine or resource.

All identities MUST be bound to both issuer and subject. A domain is discoverable identity, not automatic permission to submit requests to every user. A display name or email alone is not authorization.

## 2. Discovery

The decision provider advertises these fields in its OIDC discovery document:

| Field | Value |
|---|---|
| `openape_grant_brokering_version` | `"1.0"` |
| `openape_broker_connections_endpoint` | Owner-authenticated connection API base URL |
| `openape_brokered_grants_endpoint` | Signed broker request API URL |

The agent provider advertises:

| Field | Value |
|---|---|
| `openape_grant_brokering_version` | `"1.0"` |
| `openape_agent_domain` | Exact DNS domain used in agent subjects |
| `openape_broker_enrollment_endpoint` | Receipt-authorized agent enrollment URL |
| `openape_grants_endpoint` | Agent-facing grant API, which may mediate grants |

The broker MUST be the IdP discovered through `_ddisa.<agent-domain>`. This profile supports one exact agent domain per connection; wildcards and implicit subdomain inclusion MUST NOT be accepted. Version 1 endpoint URLs MUST use HTTPS and the respective issuer origin. This restriction is specific to this profile and does not change base Grants endpoint placement.

Implementations MUST reject redirects, non-HTTPS issuers, credentials in URLs and unsafe/private network destinations for federation discovery and backchannel calls. Isolated test configurations MAY use explicitly configured loopback issuers; such exceptions MUST NOT be advertised in production.

## 3. Human consent and connection lifecycle

Only a directly authenticated human may create or revoke a broker connection. Agent tokens, delegated human sessions and management-token shortcuts MUST NOT create consent on a human's behalf. Browser writes MUST have CSRF protection. Identity/issuer discovery MUST be verified before showing consent.

`POST {connections}` accepts `{ "broker_issuer": "https://pods.example.com", "agent_domain": "pods.example.com" }`. The owner comes exclusively from the verified human session, not the request body. The UI MUST explain that the connection permits requests, not approval, and that new agents assigned to this owner by the broker can use it. It MUST show the broker issuer and agent domain.

The decision provider returns a connection record:

```json
{
  "id": "81eec0d9-2c19-4ef9-a0cc-5cfa72253d01",
  "owner": { "issuer": "https://id.example.com", "subject": "alice@example.com" },
  "broker_issuer": "https://pods.example.com",
  "agent_domain": "pods.example.com",
  "status": "active",
  "created_at": 1789812000
}
```

`GET {connections}` lists only the authenticated owner's connections. `DELETE {connections}/{id}` revokes only that owner's connection. `POST {connections}/{id}/receipt` issues a short-lived enrollment receipt to the authenticated owner of an active connection. Connections grant only request submission, status lookup and retrieval of approved tokens for the owner's bound agents. They MUST NOT grant approval, denial, policy widening, arbitrary impersonation or delegation creation.

Revocation MUST block further create, token, approval and consume operations for every grant carrying that connection ID, including approved reusable grants. Grant history remains available to the owner. Reconnecting MUST allocate a new ID; old grants MUST NOT become valid again. Connection revocation and consumption MUST be ordered atomically at the decision provider. A command already started cannot be recalled. Outstanding one-shot requests MUST be re-created under a new connection after reconnect.

## 4. Enrollment receipt and owner binding

The receipt is an EdDSA JWT signed by the decision provider with protected `typ: "openape-broker-connection+jwt"`. Required claims are `iss`, `sub`, `aud`, `iat`, `exp`, `jti`, `connection_id`, and `agent_domain`. `iss` identifies the decision provider, `sub` its human owner, `aud` the broker issuer. Lifetime MUST NOT exceed 300 seconds.

The receipt is a bearer capability limited to enrollment under the named connection. It MUST be transported only over authenticated HTTPS and MUST NOT appear in URLs, chat, logs or rendered metadata. It is not a general human login token. The broker MUST validate signature, type, algorithm, exact audience, lifetime and DNS authority for the owner's domain, then confirm the connection is still active through the signed `connection` operation (§5). Possession is not authority to modify an existing binding.

Enrollment MUST bind each agent's immutable subject, local public-key identifier, owner issuer/subject and connection ID atomically. An existing subject MUST NOT be rebound to another owner, connection or key through enrollment. The agent's private key remains on its device. Agent enrollment details remain implementation-specific as in Core §5.3.4; a profile enrollment request carries `connection_receipt` plus the implementation's normal public-key enrollment fields.

For this version, the human subject MUST be an email-shaped DDISA identity so its decision issuer can be verified through its domain. The broker MUST NOT manufacture a human identity or mint assertions whose subject impersonates the external owner.

## 5. Authenticated broker operations

The broker authenticates the agent using its own normal agent flow, verifies the current immutable owner binding and active key, and signs a request assertion. It MUST derive the owner and connection from that binding, not from agent-supplied fields.

`POST {brokered_grants}` accepts exactly `{ "assertion": "<JWT>" }`. This dedicated endpoint is the explicit exception to base Grants §8.3's direct-caller requester assignment. Normal grant endpoints retain their existing authentication semantics.

The assertion has protected `typ: "openape-broker-request+jwt"`, EdDSA signature and these required claims:

| Claim | Meaning |
|---|---|
| `iss` | Broker issuer |
| `sub` | Agent subject; absent only for `connection` |
| `aud` | Exact decision-provider issuer |
| `iat`, `exp` | Issuance and expiry, maximum lifetime 60 seconds |
| `jti` | Unique request identifier, scoped to broker and connection |
| `connection_id` | Owner-authorized connection |
| `owner` | Human subject at the decision provider |
| `operation` | `connection`, `create`, `get`, or `token` |
| `key_id` | Agent public-key identifier; required except for `connection` |
| `request` | Original Grants request; present only for `create` |
| `grant_id` | Grant ID; present only for `get` and `token` |

The decision provider MUST validate the exact assertion type, algorithm, audience, lifetime and signature using the broker's DDISA-authorized JWKS. It MUST look up the connection, check active state, exact owner, broker issuer and agent domain, and reject unknown operation/claim combinations. The entire grant request is inside the signature. Body fields outside the assertion MUST NOT override it.

The decision provider MUST derive `requester` from the authenticated assertion's agent subject. It MUST reject owner/agent-domain mismatches. It MUST NOT create a local user for the agent. The broker's service identity MUST remain distinct from the requesting agent. This is narrowly scoped request mediation, not a management-token bypass.

Each assertion `jti` MUST be atomically accepted once. Retries with the same assertion MUST fail with `broker_request_replayed`; clients may generate a fresh assertion. If creation has an uncertain result, it MUST NOT be automatically approved or treated as successful; the agent must reconcile visible pending grants before another execution. A subsequent `get` or `token` must match the original connection, owner, agent and key. The `connection` operation returns the active record to its exact authorized broker and does not require an agent.

`create` returns a pending Grant object. This version MUST NOT apply generic auto-approval, local-user shortcuts or broad existing standing grants to brokered requests. Reusable grants require explicit human approval of that individual grant. Future policy profiles may add scoped automatic decisions.

`get` returns the matching Grant object. `token` returns the original decision-provider `{ "authz_jwt", "grant" }` only for an approved, currently authorized grant. The broker MUST NOT re-sign the returned authorization as its own decision.

## 6. Grant provenance and decision routing

Brokered Grant objects carry a decision-provider-generated `brokered` object:

```json
{
  "connection_id": "81eec0d9-2c19-4ef9-a0cc-5cfa72253d01",
  "broker_issuer": "https://pods.example.com",
  "agent_issuer": "https://pods.example.com",
  "owner": "alice@example.com",
  "key_id": "sha256-public-key-identifier"
}
```

The requester remains the agent. The owner is interpreted under the grant's decision issuer; in persisted Grant records that issuer is the storing provider. The connection ID and provenance are immutable and MUST NOT be accepted as unauthenticated input.

List, notification, detail, approval, denial, revocation, token and consume paths MUST resolve brokered ownership from this provenance and its connection rather than a local requester user row. Owner lookup MUST NOT fall back to an unrelated same-email local agent. Only the named, active human owner may approve, deny or revoke. The broker may submit/read but MUST NOT decide. The approval UI MUST distinguish owner, agent and broker, and display the actual action, target and requested duration.

The decision provider signs the ordinary AuthZ-JWT with `iss` equal to itself, `sub` equal to the external agent, and `decided_by` equal to the approving owner. It includes the complete `brokered` object. All existing audience, host, command, adapter, lifetime and grant-type rules apply. The signing rule in Grants §6.3 is unchanged. An AuthZ-JWT is an authorization statement, not an identity assertion for the agent's domain.

## 7. Execution, consumption and failures

The executor MUST have an owner-authorized local binding of human issuer/subject, agent issuer/subject/key, connection and target. Receiving a token MUST NOT create or alter this binding. It MUST verify the original decision-provider signature and exact expected issuer, agent subject, `brokered` provenance, audience, host and command/adapter binding. Trusting any issuer named by an untrusted token is forbidden.

Immediately before execution it MUST check the agent identity/key remains active with its identity provider, and consume/validate the grant directly at the decision provider over authenticated HTTPS. Consume MUST check that the provenance matches the stored grant, the human owner remains active and the broker connection is active, then atomically apply one-shot semantics. Expired, denied, revoked, used, mismatched or unavailable authority MUST prevent execution.

A plain broker response such as `{ "valid": true }` MUST NOT replace authoritative consumption. The agent-facing API may remain entirely at its broker while the executor performs this independent check internally. Signed, fully relayed consume receipts are outside version 1.

Deactivating an agent blocks authentication, broker submission and new token retrieval; the executor's active-key check blocks further execution. Revoking a connection blocks all grants using it. Every accepted create, decision, token and consumption MUST be audit-associated with owner, agent, broker, connection and grant ID. Nonces/receipts/private credentials MUST NOT be logged. Request size, rate and notification limits MUST prevent a trusted broker from creating an unbounded owner inbox.

## 8. Compatibility and errors

Implementations MUST advertise this profile only when its durable atomic connection/replay/consumption stores are configured. Existing direct/local grants retain their semantics. Clients MUST NOT silently switch a Pod's identity issuer or decision issuer. Adopting federation for an existing Pod requires a separate explicit migration; creating a new Pod may use the already authorized broker connection.

Use existing RFC 7807 errors plus `broker_connection_required`, `broker_connection_revoked`, `broker_identity_mismatch`, `broker_request_replayed`, `broker_request_invalid` and `broker_unavailable` under `https://openape.org/errors/`. Authentication errors use 401, authorization/connection errors 403, replay 409, invalid fields 400, and unavailable dependencies 503.

Conforming implementations MUST demonstrate: two distinct issuer keys; one owner consent authorizing multiple agents without local decision-provider agent accounts; rejection of foreign owners/domains, wrong audience/type/signature, assertion replay, agent/key substitution, broker self-approval and altered actions; failure after connection revocation; no revival after reconnect; exactly one successful consumption under concurrent one-shot attempts; correct owner inbox/notification routing; unchanged direct grants. These checks MUST use isolated identities and harmless commands.
