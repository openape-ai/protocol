# Error Response Examples

All error responses follow [RFC 7807](https://datatracker.ietf.org/doc/html/rfc7807) (Problem Details for HTTP APIs).

---

## DDISA Core Errors

URI base: `https://ddisa.org/errors/`

### invalid_record

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_record",
  "title": "Invalid DDISA Record",
  "status": 400,
  "detail": "DNS TXT record at _ddisa.example.com is missing the required 'idp' field."
}
```

### idp_unreachable

```http
HTTP/1.1 502 Bad Gateway
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/idp_unreachable",
  "title": "IdP Unreachable",
  "status": 502,
  "detail": "Failed to connect to https://id.example.com/.well-known/openid-configuration: connection timed out."
}
```

### invalid_manifest

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_manifest",
  "title": "Invalid Client Metadata",
  "status": 400,
  "detail": "SP metadata at https://app.example.com/.well-known/oauth-client-metadata is missing required field 'redirect_uris'."
}
```

### invalid_token

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_token",
  "title": "Invalid Token",
  "status": 401,
  "detail": "JWT signature verification failed: no matching key found in JWKS for kid 'unknown-key'."
}
```

### token_expired

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/token_expired",
  "title": "Token Expired",
  "status": 401,
  "detail": "Assertion expired at 2026-03-12T10:05:00Z (current time: 2026-03-12T10:10:00Z)."
}
```

### invalid_audience

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_audience",
  "title": "Invalid Audience",
  "status": 401,
  "detail": "Expected audience 'app.example.com' but assertion has audience 'other-app.example.com'."
}
```

### invalid_nonce

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_nonce",
  "title": "Invalid Nonce",
  "status": 401,
  "detail": "Nonce mismatch: expected 'abc123' but assertion has nonce 'xyz789'."
}
```

### unsupported_auth_method

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/unsupported_auth_method",
  "title": "Unsupported Auth Method",
  "status": 400,
  "detail": "This IdP supports ['webauthn'] but the request requires 'ed25519'."
}
```

### policy_denied

```http
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/policy_denied",
  "title": "Access Denied by Policy",
  "status": 403,
  "detail": "IdP policy mode 'deny' prevents authentication for SP 'app.example.com' at domain 'restricted.example.com'."
}
```

### invalid_pkce

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_pkce",
  "title": "Invalid PKCE",
  "status": 400,
  "detail": "Code verifier does not match code challenge (method: S256)."
}
```

### invalid_state

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://ddisa.org/errors/invalid_state",
  "title": "Invalid State",
  "status": 400,
  "detail": "State parameter mismatch. Possible CSRF attack."
}
```

---

## OpenAPE Grants Errors

URI base: `https://openape.org/errors/`

### grant_not_found

```http
HTTP/1.1 404 Not Found
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/grant_not_found",
  "title": "Grant Not Found",
  "status": 404,
  "detail": "No grant found with ID 'nonexistent-id'."
}
```

### grant_already_decided

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/grant_already_decided",
  "title": "Grant Already Decided",
  "status": 409,
  "detail": "Grant 'g1a2b3c4-...' has already been denied and cannot be approved."
}
```

### grant_expired

```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/grant_expired",
  "title": "Grant Expired",
  "status": 410,
  "detail": "Timed grant 'g1a2b3c4-...' expired at 2026-03-12T10:00:00Z."
}
```

### grant_not_approved

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/grant_not_approved",
  "title": "Grant Not Approved",
  "status": 400,
  "detail": "Grant 'g1a2b3c4-...' is in status 'pending'. A token can only be issued for approved grants."
}
```

### grant_already_used

```http
HTTP/1.1 410 Gone
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/grant_already_used",
  "title": "Grant Already Used",
  "status": 410,
  "detail": "Once grant 'g1a2b3c4-...' was consumed at 2026-03-12T10:05:00Z."
}
```

### invalid_grant_type

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/invalid_grant_type",
  "title": "Invalid Grant Type",
  "status": 400,
  "detail": "Grant type 'unlimited' is not supported. Must be one of: once, timed, always."
}
```

### missing_duration

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/missing_duration",
  "title": "Missing Duration",
  "status": 400,
  "detail": "A duration field is required when grant_type is 'timed'."
}
```

### batch_partial_failure

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/batch_partial_failure",
  "title": "Batch Partial Failure",
  "status": 207,
  "detail": "2 of 3 operations succeeded. See 'results' for details.",
  "results": [
    { "id": "grant-1", "status": "approved", "success": true },
    { "id": "grant-2", "status": "denied", "success": true },
    { "id": "grant-3", "error": "Grant is not pending: approved", "success": false }
  ]
}
```

### invalid_authz_jwt

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/problem+json

{
  "type": "https://openape.org/errors/invalid_authz_jwt",
  "title": "Invalid Authorization JWT",
  "status": 401,
  "detail": "AuthZ-JWT signature verification failed."
}
```
