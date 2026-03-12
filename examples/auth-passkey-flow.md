# WebAuthn / Passkey Authentication Flow Example

This example shows the complete OAuth 2.0 Authorization Code + PKCE flow using WebAuthn passkeys.

**Actors:**
- SP: `app.example.com`
- IdP: `https://id.example.com` (discovered via DNS)
- User: `alice@example.com`

## Step 1: SP Generates PKCE Parameters

```
code_verifier:  dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
code_challenge: E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM  (SHA-256 of verifier, base64url)
state:          xyzABC123
nonce:          n-0S6_WzA2Mj
```

## Step 2: Authorization Request (SP → IdP)

The SP redirects the user's browser to the IdP.

```http
GET /authorize?response_type=code&client_id=app.example.com&redirect_uri=https%3A%2F%2Fapp.example.com%2Fauth%2Fcallback&state=xyzABC123&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256&nonce=n-0S6_WzA2Mj&login_hint=alice%40example.com HTTP/1.1
Host: id.example.com
```

## Step 3: User Authenticates with Passkey

The IdP presents its authentication UI. The user authenticates using their WebAuthn passkey (platform authenticator or security key). This step is entirely IdP-specific and not specified by DDISA.

## Step 4: Authorization Response (IdP → SP)

After successful authentication, the IdP redirects back to the SP.

```http
HTTP/1.1 302 Found
Location: https://app.example.com/auth/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xyzABC123
```

## Step 5: SP Validates State

The SP verifies that the `state` parameter matches the value it sent in Step 2. If it does not match, the SP MUST abort the flow (potential CSRF attack).

## Step 6: Token Exchange (SP → IdP, Backchannel)

**Request:**

```http
POST /token HTTP/1.1
Host: id.example.com
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

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "assertion": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImlkcC1rZXktMjAyNi0wMSJ9.eyJpc3MiOiJodHRwczovL2lkLmV4YW1wbGUuY29tIiwic3ViIjoiYWxpY2VAZXhhbXBsZS5jb20iLCJhdWQiOiJhcHAuZXhhbXBsZS5jb20iLCJpYXQiOjE3MTAyNDQ4MDAsImV4cCI6MTcxMDI0NTEwMCwibm9uY2UiOiJuLTBTNl9XekEyTWoiLCJqdGkiOiI1NTBiNjQ5MC0xYWNlLTQ5YTEtOGE2MC0xN2UwOGQ2ZjlhYjYifQ.signature",
  "authorization_details": []
}
```

## Step 7: SP Validates Assertion

The SP decodes the JWT and performs validation.

### Decoded JWT Header

```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "idp-key-2026-01"
}
```

### Decoded JWT Payload

```json
{
  "iss": "https://id.example.com",
  "sub": "alice@example.com",
  "aud": "app.example.com",
  "iat": 1710244800,
  "exp": 1710245100,
  "nonce": "n-0S6_WzA2Mj",
  "jti": "550b6490-1ace-49a1-8a60-17e08d6f9ab6"
}
```

### Validation Checklist

| Check | Expected | Actual | Result |
|-------|----------|--------|--------|
| Signature | Valid ES256 with `kid: idp-key-2026-01` | Valid | PASS |
| `iss` | `https://id.example.com` (from DNS) | `https://id.example.com` | PASS |
| `aud` | `app.example.com` (own client_id) | `app.example.com` | PASS |
| `exp` | In the future | `1710245100` > now | PASS |
| `exp - iat` | <= 300 seconds | `300` | PASS |
| `nonce` | `n-0S6_WzA2Mj` (from Step 1) | `n-0S6_WzA2Mj` | PASS |

## Step 8: Authentication Complete

The SP creates a session for `alice@example.com` and redirects to the application.
