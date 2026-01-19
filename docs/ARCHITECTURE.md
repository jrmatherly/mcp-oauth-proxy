# Architecture Documentation

Comprehensive architecture documentation for MCP OAuth Proxy.

## Table of Contents

- [System Overview](#system-overview)
- [Architecture Diagram](#architecture-diagram)
- [Component Architecture](#component-architecture)
- [Data Flow](#data-flow)
- [Security Architecture](#security-architecture)
- [Database Design](#database-design)
- [Deployment Architecture](#deployment-architecture)

---

## System Overview

MCP OAuth Proxy is an OAuth 2.1 compliant authorization server that acts as an authentication and authorization layer for MCP (Model Context Protocol) servers. It bridges external OAuth providers (Google, Microsoft, GitHub) with MCP servers, providing user context injection and token management.

### Key Characteristics

- **OAuth 2.1 Compliance**: Full OAuth 2.1 authorization server with PKCE support
- **Provider Agnostic**: Auto-discovery support for any OAuth 2.0 provider
- **Database Flexibility**: PostgreSQL for production, SQLite for development
- **Stateless Authentication**: JWT-based tokens with cryptographic verification
- **Horizontal Scalability**: Stateless design enables multi-instance deployment

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           External OAuth Providers                   │
│              (Google, Microsoft, GitHub, Custom)                     │
└────────────────────────┬────────────────────────────────────────────┘
                         │ OAuth 2.0 Protocol
                         │ (Authorization, Token Exchange)
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        MCP OAuth Proxy Server                        │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HTTP Server Layer                         │   │
│  │  - CORS Middleware                                           │   │
│  │  - Rate Limiting                                             │   │
│  │  - Request Routing                                           │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
│                             │                                         │
│  ┌──────────────────────────┼───────────────────────────────────┐   │
│  │     OAuth 2.1 Endpoints  │  MCP Proxy Endpoint               │   │
│  │  ┌──────────────────┐    │   ┌────────────────────────┐     │   │
│  │  │ /oauth/authorize │    │   │ /mcp/* (Proxy)         │     │   │
│  │  │ /oauth/token     │    │   │ - Token Validation     │     │   │
│  │  │ /oauth/callback  │    │   │ - Header Injection     │     │   │
│  │  │ /oauth/revoke    │    │   │ - Request Forwarding   │     │   │
│  │  │ /oauth/register  │    │   └────────────────────────┘     │   │
│  │  └──────────────────┘    │                                   │   │
│  └──────────────────────────┴───────────────────────────────────┘   │
│                             │                                         │
│  ┌──────────────────────────┴───────────────────────────────────┐   │
│  │                     Core Components                           │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐    │   │
│  │  │ Provider    │  │ Token        │  │ Database         │    │   │
│  │  │ Manager     │  │ Manager      │  │ Store            │    │   │
│  │  │ - Discovery │  │ - JWT Gen    │  │ - GORM          │    │   │
│  │  │ - JWKS      │  │ - Validation │  │ - PostgreSQL    │    │   │
│  │  └─────────────┘  └──────────────┘  │ - SQLite        │    │   │
│  │                                      └──────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────────────┘
                         │ Authenticated Requests
                         │ (with User Context Headers)
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          MCP Server                                  │
│                   (Gmail, Slack, Custom Tools)                       │
│  Headers Received:                                                   │
│  - X-Forwarded-User: user_id                                        │
│  - X-Forwarded-Email: user@example.com                              │
│  - X-Forwarded-Name: John Doe                                       │
│  - X-Forwarded-Access-Token: provider_token                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

### 1. HTTP Server Layer (`pkg/proxy`)

**Responsibility:** Request handling, routing, middleware

**Key Components:**

- `OAuthProxy` - Main server struct
- CORS middleware
- Rate limiting middleware
- Route setup and handler registration

**Files:**

- [`pkg/proxy/proxy.go`](../pkg/proxy/proxy.go)

**Features:**

- HTTP server lifecycle management
- Middleware chain execution
- Health check endpoint
- Graceful shutdown

---

### 2. OAuth 2.1 Handlers (`pkg/oauth/*`)

#### Authorization Handler (`pkg/oauth/authorize`)

**Responsibility:** Initiate OAuth authorization flow

**Flow:**

1. Validate client and request parameters
2. Check PKCE requirements for public clients
3. Store authorization request in database
4. Redirect to external OAuth provider

**Files:**

- [`pkg/oauth/authorize/authorize.go`](../pkg/oauth/authorize/authorize.go)

**Interface:**

```go
type AuthorizationStore interface {
    GetClient(clientID string) (*types.ClientInfo, error)
    StoreAuthRequest(key string, data map[string]any) error
}
```

---

#### Token Handler (`pkg/oauth/token`)

**Responsibility:** Issue and refresh access tokens

**Grant Types:**

1. **Authorization Code Grant** - Exchange code for tokens
2. **Refresh Token Grant** - Refresh access token

**Flow (Authorization Code):**

1. Validate client credentials
2. Validate authorization code and PKCE
3. Exchange code with external provider
4. Generate JWT access token
5. Store token and grant in database

**Files:**

- [`pkg/oauth/token/token.go`](../pkg/oauth/token/token.go)

**Interface:**

```go
type TokenStore interface {
    GetClient(clientID string) (*types.ClientInfo, error)
    StoreToken(token *types.TokenData) error
    ValidateAuthCode(code string) (string, string, error)
    GetGrant(grantID string, userID string) (*types.Grant, error)
    DeleteAuthCode(code string) error
    GetTokenByRefreshToken(refreshToken string) (*types.TokenData, error)
    RevokeToken(token string) error
}
```

---

#### Callback Handler (`pkg/oauth/callback`)

**Responsibility:** Handle OAuth provider callback

**Flow:**

1. Receive authorization code from provider
2. Validate state parameter
3. Exchange code for provider tokens
4. Fetch user information
5. Create grant and authorization code
6. Redirect to client with code

**Files:**

- [`pkg/oauth/callback/callback.go`](../pkg/oauth/callback/callback.go)

---

#### Revocation Handler (`pkg/oauth/revoke`)

**Responsibility:** Revoke access and refresh tokens

**Files:**

- [`pkg/oauth/revoke/revoke.go`](../pkg/oauth/revoke/revoke.go)

---

#### Registration Handler (`pkg/oauth/register`)

**Responsibility:** Dynamic client registration (RFC 7591)

**Files:**

- [`pkg/oauth/register/register.go`](../pkg/oauth/register/register.go)

---

### 3. Provider Manager (`pkg/providers`)

**Responsibility:** OAuth provider integration and management

**Key Components:**

- `Provider` interface - Abstract OAuth provider operations
- `GenericProvider` - Auto-discovery implementation
- `ProviderManager` - Multi-provider management

**Files:**

- [`pkg/providers/provider.go`](../pkg/providers/provider.go)
- [`pkg/providers/generic.go`](../pkg/providers/generic.go)
- [`pkg/providers/manager.go`](../pkg/providers/manager.go)

**Interface:**

```go
type Provider interface {
    GetName() string
    GetAuthorizationURL(clientID, redirectURI, scope, state string) string
    ExchangeCodeForToken(ctx context.Context, code, clientID, clientSecret, redirectURI string) (*TokenInfo, error)
    GetUserInfo(ctx context.Context, accessToken string) (*UserInfo, error)
    RefreshToken(ctx context.Context, refreshToken, clientID, clientSecret string) (*TokenInfo, error)
    GetJWKS() (map[string]interface{}, error)
}
```

**Features:**

- OAuth 2.0 auto-discovery (`.well-known/openid-configuration`)
- JWKS fetching and caching
- Multi-provider support with concurrent access
- Mock provider for testing

---

### 4. Token Manager (`pkg/tokens`)

**Responsibility:** JWT token generation and validation

**Key Components:**

- `TokenManager` - JWT operations
- API key authentication support

**Files:**

- [`pkg/tokens/manager.go`](../pkg/tokens/manager.go)
- [`pkg/tokens/apikey.go`](../pkg/tokens/apikey.go)

**Features:**

- RS256 JWT generation
- Token validation against JWKS
- Custom claims injection
- API key generation and validation

**JWT Claims:**

```go
{
    "iss": "http://localhost:8080",
    "sub": "user_id",
    "aud": ["client_id"],
    "exp": 1234567890,
    "iat": 1234567890,
    "scope": "openid email profile",
    "email": "user@example.com",
    "name": "John Doe",
    "grant_id": "grant_uuid"
}
```

---

### 5. Database Store (`pkg/db`)

**Responsibility:** Data persistence and schema management

**Supported Databases:**

- PostgreSQL (production)
- SQLite (development/testing)

**Files:**

- [`pkg/db/db.go`](../pkg/db/db.go)

**Schema Components:**

- `ClientInfo` - OAuth client registrations
- `Grant` - User authorization grants
- `TokenData` - Issued tokens
- `AuthorizationCode` - Authorization codes
- `StoredAuthRequest` - Pending auth requests

**Features:**

- GORM ORM abstraction
- Automatic schema migrations
- Token encryption at rest
- Expired data cleanup

**Operations:**

```go
type Store struct {
    db     *gorm.DB
    dbType string
}

// Client operations
GetClient(clientID string) (*types.ClientInfo, error)
StoreClient(client *types.ClientInfo) error

// Grant operations
StoreGrant(grant *types.Grant) error
UpdateGrant(grant *types.Grant) error
GetGrant(grantID string, userID string) (*types.Grant, error)

// Token operations
StoreToken(token *types.TokenData) error
GetToken(token string) (*types.TokenData, error)
RevokeToken(token string) error

// Auth code operations
StoreAuthCode(code, grantID, userID string) error
ValidateAuthCode(code string) (string, string, error)
DeleteAuthCode(code string) error
```

---

### 6. Type Definitions (`pkg/types`)

**Responsibility:** Shared data structures

**Files:**

- [`pkg/types/types.go`](../pkg/types/types.go) - Core types
- [`pkg/types/db_types.go`](../pkg/types/db_types.go) - Database models
- [`pkg/types/oauth_types.go`](../pkg/types/oauth_types.go) - OAuth structures
- [`pkg/types/apikey_types.go`](../pkg/types/apikey_types.go) - API key types

**Key Types:**

- `Config` - Server configuration
- `ClientInfo` - OAuth client metadata
- `Grant` - User authorization grant
- `TokenData` - Issued token information
- `AuthRequest` - Authorization request parameters
- `OAuthError` - Standardized error responses

---

### 7. Supporting Components

#### Encryption (`pkg/encryption`)

**Files:**

- [`pkg/encryption/encryption.go`](../pkg/encryption/encryption.go)
- [`pkg/encryption/random.go`](../pkg/encryption/random.go)

**Features:**

- AES-256-GCM encryption for refresh tokens
- Secure random value generation
- Code challenge generation for PKCE

---

#### Handler Utilities (`pkg/handlerutils`)

**Files:**

- [`pkg/handlerutils/handlerutils.go`](../pkg/handlerutils/handlerutils.go)

**Features:**

- JSON response helpers
- Error response formatting
- Request parsing utilities

---

#### Rate Limiting (`pkg/ratelimit`)

**Files:**

- [`pkg/ratelimit/ratelimiter.go`](../pkg/ratelimit/ratelimiter.go)

**Features:**

- IP-based rate limiting
- Configurable limits per endpoint
- Token bucket algorithm

---

## Data Flow

### Authorization Code Flow

```
1. Client → /oauth/authorize
   ├─ Validate client and parameters
   ├─ Store auth request in DB
   └─ Redirect to external OAuth provider

2. User authenticates at provider
   └─ Provider redirects to /oauth/callback

3. Proxy → /oauth/callback
   ├─ Validate state parameter
   ├─ Exchange code with provider
   ├─ Fetch user information
   ├─ Create grant in DB
   ├─ Create authorization code in DB
   └─ Redirect to client with code

4. Client → /oauth/token
   ├─ Validate authorization code
   ├─ Validate PKCE (if applicable)
   ├─ Generate JWT access token
   ├─ Encrypt refresh token
   ├─ Store token in DB
   └─ Return tokens to client

5. Client → /mcp/* (with Bearer token)
   ├─ Validate JWT signature
   ├─ Extract user claims
   ├─ Inject user context headers
   └─ Proxy to MCP server
```

---

### Refresh Token Flow

```
1. Client → /oauth/token (grant_type=refresh_token)
   ├─ Validate refresh token
   ├─ Decrypt refresh token
   ├─ Validate client credentials
   ├─ Check token expiration
   ├─ Get grant from DB
   ├─ Exchange with provider (if needed)
   ├─ Generate new JWT access token
   ├─ Update token in DB
   └─ Return new access token
```

---

## Security Architecture

### Token Security

**Access Tokens (JWT):**

- Algorithm: RS256 (RSA-SHA256)
- Expiration: 1 hour (configurable)
- Signed with private RSA key
- Validated against JWKS endpoint

**Refresh Tokens:**

- AES-256-GCM encryption at rest
- Stored encrypted in database
- Single-use pattern (rotated on refresh)
- Longer expiration (90 days)

### PKCE (Proof Key for Code Exchange)

**Required for public clients** (`token_endpoint_auth_method: "none"`):

1. Client generates random `code_verifier` (43-128 chars)
2. Client generates `code_challenge = BASE64URL(SHA256(code_verifier))`
3. Authorization request includes `code_challenge` and `code_challenge_method=S256`
4. Token request includes `code_verifier`
5. Server validates: `code_challenge == BASE64URL(SHA256(code_verifier))`

**Implementation:** [`pkg/db/db.go:GenerateCodeChallenge`](../pkg/db/db.go)

### Client Authentication

**Methods Supported:**

1. `client_secret_post` - Credentials in request body
2. `client_secret_basic` - HTTP Basic Authentication
3. `none` - Public clients (PKCE required)

### Data Encryption

**Encryption Key:**

- 32-byte AES key (256-bit)
- Base64-encoded environment variable
- Used for refresh token encryption

**Encrypted Data:**

- Refresh tokens in database
- OAuth provider access tokens in grants

---

## Database Design

See [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md) for complete schema documentation.

**Tables:**

- `client_infos` - OAuth client registrations
- `grants` - User authorization grants
- `token_datas` - Issued access/refresh tokens
- `authorization_codes` - Authorization codes
- `stored_auth_requests` - Pending authorization requests

**Relationships:**

```
ClientInfo (1) ─── (N) Grant
Grant (1) ─── (N) TokenData
Grant (1) ─── (N) AuthorizationCode
```

---

## Deployment Architecture

### Single Instance Deployment

```
┌──────────────────────┐
│   Load Balancer      │
│   (Optional)         │
└──────────┬───────────┘
           │
┌──────────▼───────────┐
│  MCP OAuth Proxy     │
│  Port: 8080          │
└──────────┬───────────┘
           │
┌──────────▼───────────┐
│  PostgreSQL/SQLite   │
│  Database            │
└──────────────────────┘
```

### Multi-Instance Deployment (Horizontal Scale)

```
                    ┌──────────────────────┐
                    │   Load Balancer      │
                    └─────┬───────┬────────┘
                          │       │
              ┌───────────┘       └───────────┐
              │                               │
    ┌─────────▼────────┐          ┌─────────▼────────┐
    │ MCP OAuth Proxy  │          │ MCP OAuth Proxy  │
    │ Instance 1       │          │ Instance 2       │
    └─────────┬────────┘          └─────────┬────────┘
              │                               │
              └───────────┬───────────────────┘
                          │
                  ┌───────▼───────┐
                  │  PostgreSQL   │
                  │  (Shared DB)  │
                  └───────────────┘
```

**Requirements for Horizontal Scaling:**

- Shared PostgreSQL database (SQLite not suitable)
- Stateless design (no server-side sessions)
- Load balancer with sticky sessions (optional)
- Shared encryption key across instances

### Docker Deployment

**Container:** `ghcr.io/obot-platform/mcp-oauth-proxy:latest`

**Environment Variables:**

- `OAUTH_CLIENT_ID` - OAuth client ID
- `OAUTH_CLIENT_SECRET` - OAuth client secret
- `OAUTH_AUTHORIZE_URL` - Provider authorization URL
- `SCOPES_SUPPORTED` - Comma-separated scopes
- `MCP_SERVER_URL` - Backend MCP server URL
- `DATABASE_DSN` - PostgreSQL connection string
- `ENCRYPTION_KEY` - Base64-encoded AES key

**Docker Compose:** See [`docker-compose.yml`](../docker-compose.yml)

---

## Cross-References

- **API Reference:** [API_REFERENCE.md](API_REFERENCE.md)
- **Database Schema:** [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md)
- **Development Guide:** [DEVELOPMENT.md](DEVELOPMENT.md)
- **Testing Guide:** [../TESTING.md](../TESTING.md)
- **Main README:** [../README.md](../README.md)
