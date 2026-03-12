# Discovery Flow Example

This example shows the complete DDISA discovery process for `alice@example.com`.

## Step 1: DNS TXT Record Query

Query the `_ddisa.example.com` TXT record.

### Using dig

```
$ dig TXT _ddisa.example.com +short
"v=ddisa1 idp=https://id.example.com; mode=open; priority=10"
```

### Using DNS-over-HTTPS (Cloudflare)

**Request:**

```http
GET https://cloudflare-dns.com/dns-query?type=TXT&name=_ddisa.example.com HTTP/1.1
Accept: application/dns-json
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/dns-json

{
  "Status": 0,
  "TC": false,
  "RD": true,
  "RA": true,
  "AD": true,
  "CD": false,
  "Question": [
    { "name": "_ddisa.example.com.", "type": 16 }
  ],
  "Answer": [
    {
      "name": "_ddisa.example.com.",
      "type": 16,
      "TTL": 300,
      "data": "\"v=ddisa1 idp=https://id.example.com; mode=open; priority=10\""
    }
  ]
}
```

### Parsed Record

```json
{
  "version": "ddisa1",
  "idp": "https://id.example.com",
  "mode": "open",
  "priority": 10
}
```

## Step 2: Fetch IdP OIDC Discovery

**Request:**

```http
GET https://id.example.com/.well-known/openid-configuration HTTP/1.1
Accept: application/json
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "issuer": "https://id.example.com",
  "authorization_endpoint": "https://id.example.com/authorize",
  "token_endpoint": "https://id.example.com/token",
  "jwks_uri": "https://id.example.com/.well-known/jwks.json",
  "response_types_supported": ["code"],
  "code_challenge_methods_supported": ["S256"],
  "ddisa_version": "1.0",
  "ddisa_auth_methods_supported": ["webauthn", "ed25519"],
  "ddisa_agent_challenge_endpoint": "https://id.example.com/api/agent/challenge",
  "ddisa_agent_authenticate_endpoint": "https://id.example.com/api/agent/authenticate",
  "openape_grants_endpoint": "https://id.example.com/api/grants",
  "openape_grant_types_supported": ["once", "timed", "always"],
  "openape_grant_categories_supported": ["command", "delegation"]
}
```

## Step 3: Fetch IdP JWKS

**Request:**

```http
GET https://id.example.com/.well-known/jwks.json HTTP/1.1
Accept: application/json
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: public, max-age=3600

{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
      "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
      "alg": "ES256",
      "use": "sig",
      "kid": "idp-key-2026-01"
    }
  ]
}
```

## Step 4: Fetch SP Client Metadata (Optional)

**Request:**

```http
GET https://app.example.com/.well-known/oauth-client-metadata HTTP/1.1
Accept: application/json
```

**Response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_id": "app.example.com",
  "client_name": "Example App",
  "redirect_uris": ["https://app.example.com/auth/callback"],
  "client_uri": "https://app.example.com",
  "logo_uri": "https://app.example.com/logo.png",
  "contacts": ["admin@example.com"],
  "policy_uri": "https://app.example.com/privacy",
  "tos_uri": "https://app.example.com/terms"
}
```
