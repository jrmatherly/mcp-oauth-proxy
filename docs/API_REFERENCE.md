# API Reference

Complete API reference for MCP OAuth Proxy endpoints.

## Table of Contents

1. [API Reference](#api-reference)
   1. [Table of Contents](#table-of-contents)
   2. [OAuth 2.1 Endpoints](#oauth-21-endpoints)
      1. [Authorization Endpoint](#authorization-endpoint)
      2. [Token Endpoint](#token-endpoint)
         1. [Authorization Code Grant](#authorization-code-grant)
         2. [Refresh Token Grant](#refresh-token-grant)
      3. [Callback Endpoint](#callback-endpoint)
      4. [Revocation Endpoint](#revocation-endpoint)
      5. [Registration Endpoint](#registration-endpoint)
   3. [Metadata Endpoints](#metadata-endpoints)
      1. [OAuth Metadata](#oauth-metadata)
      2. [Protected Resource Metadata](#protected-resource-metadata)
   4. [MCP Proxy Endpoints](#mcp-proxy-endpoints)
      1. [MCP Proxy Handler](#mcp-proxy-handler)
   5. [Health \& Monitoring](#health--monitoring)
      1. [Health Check](#health-check)
   6. [Authentication Methods](#authentication-methods)
      1. [Client Authentication](#client-authentication)
      2. [PKCE (Proof Key for Code Exchange)](#pkce-proof-key-for-code-exchange)
   7. [Rate Limiting](#rate-limiting)
   8. [CORS Support](#cors-support)
   9. [Error Response Format](#error-response-format)
      1. [Common Error Codes](#common-error-codes)
   10. [Cross-References](#cross-references)

---

## OAuth 2.1 Endpoints

### Authorization Endpoint

**Endpoint:** `GET|POST /oauth/authorize`

**Description:** Initiates the OAuth 2.1 authorization flow. Redirects to the external OAuth provider for user authentication.

**Request Parameters:**

| Parameter | Type | Required | Description |
| ----------- | ------ | ---------- | ------------- |
| `response_type` | string | Yes | Must be `code` for authorization code grant |
| `client_id` | string | Yes | OAuth client identifier |
| `redirect_uri` | string | Yes | Callback URL after authorization |
| `scope` | string | No | Space-separated scopes (defaults to supported scopes) |
| `state` | string | No | Opaque value for CSRF protection |
| `code_challenge` | string | Conditional | PKCE code challenge (required for public clients) |
| `code_challenge_method` | string | Conditional | Must be `S256` (SHA-256 hash) |

**Response:**

Redirects to external OAuth provider authorization URL.

**Error Responses:**

```json
{
  "error": "invalid_request",
  "error_description": "Missing required parameters"
}
```

**Error Codes:**

- `invalid_request` - Missing or malformed parameters
- `unauthorized_client` - Client not authorized
- `unsupported_response_type` - Response type not supported
- `invalid_scope` - Requested scope invalid

**Implementation:** [`pkg/oauth/authorize/authorize.go`](../pkg/oauth/authorize/authorize.go)

**Example Request:**

```http
GET /oauth/authorize?response_type=code&client_id=abc123&redirect_uri=http://localhost:3000/callback&code_challenge=E9Melhoa...&code_challenge_method=S256 HTTP/1.1
Host: localhost:8080
```

---

### Token Endpoint

**Endpoint:** `POST /oauth/token`

**Description:** Exchanges authorization code for access token or refreshes an existing token.

**Request Parameters (application/x-www-form-urlencoded):**

#### Authorization Code Grant

| Parameter | Type | Required | Description |
| ----------- | ------ | ---------- | ------------- |
| `grant_type` | string | Yes | Must be `authorization_code` |
| `code` | string | Yes | Authorization code from callback |
| `redirect_uri` | string | Yes | Must match authorization request |
| `client_id` | string | Yes | OAuth client identifier |
| `client_secret` | string | Conditional | Required for confidential clients |
| `code_verifier` | string | Conditional | PKCE verifier (if code_challenge used) |

#### Refresh Token Grant

| Parameter | Type | Required | Description |
| ----------- | ------ | ---------- | ------------- |
| `grant_type` | string | Yes | Must be `refresh_token` |
| `refresh_token` | string | Yes | Valid refresh token |
| `client_id` | string | Yes | OAuth client identifier |
| `client_secret` | string | Conditional | Required for confidential clients |
| `scope` | string | No | Requested scope (cannot exceed original) |

**Response:**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "encrypted_refresh_token",
  "scope": "openid email profile"
}
```

**Error Responses:**

```json
{
  "error": "invalid_grant",
  "error_description": "Authorization code is invalid or expired"
}
```

**Error Codes:**

- `invalid_request` - Malformed request
- `invalid_client` - Client authentication failed
- `invalid_grant` - Authorization code/refresh token invalid
- `unauthorized_client` - Client not authorized for grant type
- `unsupported_grant_type` - Grant type not supported

**Implementation:** [`pkg/oauth/token/token.go`](../pkg/oauth/token/token.go)

**Example Request:**

```http
POST /oauth/token HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=xyz789&redirect_uri=http://localhost:3000/callback&client_id=abc123&code_verifier=dBjftJeZ...
```

---

### Callback Endpoint

**Endpoint:** `GET /oauth/callback`

**Description:** Handles OAuth provider callback after user authentication. Internal endpoint used during authorization flow.

**Request Parameters:**

| Parameter | Type | Required | Description |
| ----------- | ------ | ---------- | ------------- |
| `code` | string | Yes | Authorization code from provider |
| `state` | string | Yes | State parameter for CSRF validation |
| `error` | string | No | Error code if authorization failed |
| `error_description` | string | No | Human-readable error description |

**Response:**

Redirects to success page or client redirect URI with authorization code.

**Implementation:** [`pkg/oauth/callback/callback.go`](../pkg/oauth/callback/callback.go)

---

### Revocation Endpoint

**Endpoint:** `POST /oauth/revoke`

**Description:** Revokes an access token or refresh token.

**Request Parameters (application/x-www-form-urlencoded):**

| Parameter | Type | Required | Description |
| ----------- | ------ | ---------- | ------------- |
| `token` | string | Yes | Token to revoke |
| `token_type_hint` | string | No | `access_token` or `refresh_token` |
| `client_id` | string | Yes | OAuth client identifier |
| `client_secret` | string | Conditional | Required for confidential clients |

**Response:**

```http
HTTP/1.1 200 OK
```

Empty response body on success (RFC 7009 specification).

**Error Responses:**

```json
{
  "error": "invalid_client",
  "error_description": "Client authentication failed"
}
```

**Implementation:** [`pkg/oauth/revoke/revoke.go`](../pkg/oauth/revoke/revoke.go)

**Example Request:**

```http
POST /oauth/revoke HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

token=eyJhbGciOiJSUzI1NiIs...&client_id=abc123&client_secret=secret123
```

---

### Registration Endpoint

**Endpoint:** `POST /oauth/register`

**Description:** Dynamic client registration endpoint (RFC 7591).

**Request Body (application/json):**

| Field | Type | Required | Description |
| ------- | ------ | ---------- | ------------- |
| `client_name` | string | No | Human-readable client name |
| `redirect_uris` | []string | Yes | Array of redirect URIs |
| `token_endpoint_auth_method` | string | No | `client_secret_post`, `client_secret_basic`, or `none` |
| `grant_types` | []string | No | Array of grant types (defaults to `authorization_code`) |
| `response_types` | []string | No | Array of response types (defaults to `code`) |
| `scope` | string | No | Space-separated scopes |

**Response:**

```json
{
  "client_id": "generated_client_id",
  "client_secret": "generated_client_secret",
  "client_name": "My Application",
  "redirect_uris": ["http://localhost:3000/callback"],
  "token_endpoint_auth_method": "client_secret_post",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "scope": "openid email profile"
}
```

**Error Responses:**

```json
{
  "error": "invalid_client_metadata",
  "error_description": "redirect_uris is required"
}
```

**Implementation:** [`pkg/oauth/register/register.go`](../pkg/oauth/register/register.go)

**Example Request:**

```http
POST /oauth/register HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "client_name": "My Application",
  "redirect_uris": ["http://localhost:3000/callback"],
  "token_endpoint_auth_method": "none"
}
```

---

## Metadata Endpoints

### OAuth Metadata

**Endpoint:** `GET /.well-known/oauth-authorization-server`

**Description:** OAuth 2.0 Authorization Server Metadata (RFC 8414).

**Response:**

```json
{
  "issuer": "http://localhost:8080",
  "authorization_endpoint": "http://localhost:8080/oauth/authorize",
  "token_endpoint": "http://localhost:8080/oauth/token",
  "revocation_endpoint": "http://localhost:8080/oauth/revoke",
  "registration_endpoint": "http://localhost:8080/oauth/register",
  "jwks_uri": "http://localhost:8080/.well-known/jwks.json",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic", "none"]
}
```

**Implementation:** [`pkg/proxy/proxy.go:oauthMetadataHandler`](../pkg/proxy/proxy.go)

---

### Protected Resource Metadata

**Endpoint:** `GET /.well-known/oauth-protected-resource`

**Description:** OAuth 2.0 Protected Resource Metadata (RFC 8705).

**Response:**

```json
{
  "resource": "http://localhost:8080/mcp",
  "authorization_servers": ["http://localhost:8080"],
  "scopes_supported": ["openid", "email", "profile"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "http://localhost:8080/docs"
}
```

**Implementation:** [`pkg/proxy/proxy.go:protectedResourceMetadataHandler`](../pkg/proxy/proxy.go)

---

## MCP Proxy Endpoints

### MCP Proxy Handler

**Endpoint:** `POST /mcp/*` (or custom prefix)

**Description:** Proxies authenticated MCP requests to the backend MCP server. Validates bearer token and injects user context headers.

**Request Headers:**

| Header | Type | Required | Description |
| -------- | ------ | ---------- | ------------- |
| `Authorization` | string | Yes | `Bearer <access_token>` |
| `Content-Type` | string | Yes | `application/json` |

**Injected Headers (to MCP Server):**

| Header | Description | Example |
| -------- | ------------- | --------- |
| `X-Forwarded-User` | User ID from OAuth provider | `12345678901234567890` |
| `X-Forwarded-Email` | User's email address | `user@example.com` |
| `X-Forwarded-Name` | User's display name | `John Doe` |
| `X-Forwarded-Access-Token` | OAuth access token for API calls | `ya29.a0ARrdaM...` |

**Request Body:**

MCP JSON-RPC request (passed through to backend).

**Response:**

MCP JSON-RPC response from backend server.

**Error Responses:**

```json
{
  "error": "invalid_token",
  "error_description": "The access token is invalid or expired"
}
```

**Error Codes:**

- `invalid_token` - Token invalid, expired, or malformed
- `insufficient_scope` - Token lacks required scopes

**Implementation:** [`pkg/proxy/proxy.go:mcpProxyHandler`](../pkg/proxy/proxy.go)

**Example Request:**

```http
POST /mcp/gmail HTTP/1.1
Host: localhost:8080
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "id": 1
}
```

---

## Health & Monitoring

### Health Check

**Endpoint:** `GET /health`

**Description:** Health check endpoint for monitoring and load balancers.

**Response:**

```json
{
  "status": "healthy",
  "timestamp": "2026-01-15T19:57:45Z"
}
```

**Implementation:** [`pkg/proxy/proxy.go:healthHandler`](../pkg/proxy/proxy.go)

---

## Authentication Methods

### Client Authentication

MCP OAuth Proxy supports three client authentication methods:

1. **`client_secret_post`** - Client credentials in request body
2. **`client_secret_basic`** - HTTP Basic Authentication
3. **`none`** - Public clients (PKCE required)

### PKCE (Proof Key for Code Exchange)

Required for public clients (`token_endpoint_auth_method: "none"`):

1. Generate code verifier: 43-128 character random string
2. Generate code challenge: `BASE64URL(SHA256(code_verifier))`
3. Send `code_challenge` and `code_challenge_method=S256` in authorization request
4. Send `code_verifier` in token request

---

## Rate Limiting

All endpoints are protected by rate limiting:

- **Default:** 100 requests per minute per IP
- **Header:** `X-RateLimit-Remaining` indicates remaining requests
- **Error:** Returns `429 Too Many Requests` when exceeded

---

## CORS Support

All endpoints support CORS with the following headers:

- `Access-Control-Allow-Origin: *`
- `Access-Control-Allow-Methods: GET, POST, OPTIONS`
- `Access-Control-Allow-Headers: Authorization, Content-Type`
- `Access-Control-Max-Age: 86400`

---

## Error Response Format

All errors follow OAuth 2.0 error response format:

```json
{
  "error": "error_code",
  "error_description": "Human-readable description",
  "error_uri": "https://tools.ietf.org/html/rfc6749#section-X.X"
}
```

### Common Error Codes

| Error Code | HTTP Status | Description |
| ----------- | ------------- | ------------- |
| `invalid_request` | 400 | Request missing required parameter |
| `invalid_client` | 401 | Client authentication failed |
| `invalid_grant` | 400 | Authorization grant is invalid |
| `unauthorized_client` | 400 | Client not authorized |
| `unsupported_grant_type` | 400 | Grant type not supported |
| `invalid_scope` | 400 | Requested scope invalid |
| `invalid_token` | 401 | Access token invalid or expired |
| `insufficient_scope` | 403 | Token lacks required scope |

---

## Cross-References

- **Architecture:** See [ARCHITECTURE.md](ARCHITECTURE.md) for system design
- **Development:** See [DEVELOPMENT.md](DEVELOPMENT.md) for setup guide
- **Testing:** See [TESTING.md](../TESTING.md) for test documentation
- **Database Schema:** See [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md) for data models
