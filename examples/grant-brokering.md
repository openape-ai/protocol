# Grant brokering example

The [decoded JSON examples](grant-brokering.json) show distinct agent and decision issuers. Each broker operation is a separate JWT and MUST use a fresh `jti` and current timestamps in a real exchange; the shared illustrative values are not reusable credentials. Sign receipts with `openape-broker-connection+jwt` and broker requests with `openape-broker-request+jwt` as the protected type. The authorization token remains signed by the decision provider. The example grants a harmless synthetic printf command; production CLI adapters supply canonical structured permissions.

Sequence: human consent → enrollment receipt → broker connection verification → local key enrollment → signed create → owner approval → signed token retrieval → executor signature/binding checks → authoritative consume → harmless command. Replaying a broker assertion fails; revoking the connection prevents subsequent token retrieval and consume even if the JWT has not expired.
