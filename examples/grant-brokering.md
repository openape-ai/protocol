# Grant brokering example

The [decoded JSON examples](grant-brokering.json) show distinct agent and decision issuers. Each broker operation is a separate JWT and MUST use a fresh `jti` and current timestamps in a real exchange; the shared illustrative values are not reusable credentials. Sign receipts with `openape-broker-connection+jwt` and broker requests with `openape-broker-request+jwt` as the protected type. The authorization token remains signed by the decision provider. The example grants a harmless synthetic printf command; production CLI adapters supply canonical structured permissions.

Sequence: human consent → enrollment receipt → broker connection verification → local key enrollment → signed create → owner approval → signed token retrieval → executor signature/binding checks → authoritative consume → harmless command. Replaying a broker assertion fails; revoking the connection prevents subsequent token retrieval and consume even if the JWT has not expired.

The decision provider's illustrated discovery extension enables `requester_name` on the four signed agent operations. The broker obtained `Invoice Assistant` from its already authenticated registered agent, not from the nested request. The returned Grant stores it as `brokered.requester_name`; the authorization example deliberately omits it. A later matching signed operation may refresh the stored display name without changing grant state or identity bindings.

For a legacy decision provider that omits the capability or advertises `false`, omit `requester_name` from all assertions. The stable requester identity and authorization behavior are unchanged; the UI uses its honest missing-name fallback. `connection` assertions never carry an agent display name.

The `authorize` example illustrates the already implemented per-call path with program hash and argv, not caller-supplied permission details. It carries the same verified display name, while preserving the existing connection/agent/key binding and human decision requirement.
