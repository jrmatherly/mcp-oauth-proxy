# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP OAuth Proxy is an OAuth 2.1 authorization server that sits in front of MCP (Model Context Protocol) servers, providing authentication via external OAuth providers (Google, Microsoft, GitHub). It handles the complete OAuth flow and injects user context headers (`X-Forwarded-User`, `X-Forwarded-Email`, `X-Forwarded-Name`, `X-Forwarded-Access-Token`) when proxying requests to backend MCP servers.

**Key Technologies:** Go 1.25.0, GORM (PostgreSQL/SQLite), JWT (RS256), OAuth 2.1 with PKCE

## Development Commands

### Testing

```bash
# Fast unit tests (use for quick feedback)
make test-short
go test -v -short ./...

# Run specific test
go test -v -run TestTokenEndpoint ./pkg/oauth/token/

# All tests (includes integration tests)
make test

# Integration tests (requires PostgreSQL)
make setup-test-db  # First time only
make test-integration

# Race detection
make test-race

# Coverage report
make test-coverage  # Generates coverage.html
```

### Building & Running

```bash
# Build binary
make build  # Outputs to bin/mcp-oauth-proxy

# Quick demo with SQLite
./demo_sqlite.sh

# Run locally with environment variables
export OAUTH_CLIENT_ID="your-client-id"
export OAUTH_CLIENT_SECRET="your-client-secret"
export OAUTH_AUTHORIZE_URL="https://accounts.google.com"
export SCOPES_SUPPORTED="openid,email,profile"
export MCP_SERVER_URL="http://localhost:9000/mcp/gmail"
export ENCRYPTION_KEY=$(openssl rand -base64 32)
go run main.go
```

### Code Quality

```bash
make fmt    # Format code
make vet    # Go vet
make lint   # golangci-lint (must be installed)
make ci     # Run all checks (fmt, vet, lint, test-short, test-race, test-coverage)
```

## Architecture

### Request Flow

**OAuth Authorization Flow:**

1. Client → `/oauth/authorize` (pkg/oauth/authorize) → Redirects to external provider
2. Provider → `/oauth/callback` (pkg/oauth/callback) → Exchanges code, creates grant
3. Client → `/oauth/token` (pkg/oauth/token) → Issues JWT access token + encrypted refresh token
4. Client → `/mcp/*` (pkg/proxy) → Validates JWT, injects headers, proxies to MCP server

**Critical Flow Details:**

- **PKCE is required** for public clients (`token_endpoint_auth_method: "none"`)
- Authorization codes are single-use and expire in 10 minutes
- Access tokens are RS256 JWTs (1 hour expiry)
- Refresh tokens are AES-256-GCM encrypted (90 days expiry)
- Provider access tokens are stored encrypted in grants table

### Component Responsibilities

**pkg/proxy/proxy.go - `OAuthProxy`**

- Main HTTP server with CORS and rate limiting middleware
- Routes all OAuth endpoints and MCP proxy
- **Key method:** `mcpProxyHandler()` - validates JWT, extracts user claims, injects headers

**pkg/oauth/** - OAuth 2.1 Endpoint Handlers

- Each subdirectory (authorize, token, callback, revoke, register) has its own handler
- Handlers define minimal interfaces (e.g., `TokenStore`, `AuthorizationStore`) for database operations
- **Pattern:** `NewHandler(deps) http.Handler` returns handler with injected dependencies

**pkg/db/db.go - `Store`**

- GORM-based database abstraction supporting PostgreSQL and SQLite
- **Auto-detects database type:** Empty DSN → SQLite (`data/oauth_proxy.db`), otherwise PostgreSQL
- **Auto-migration:** `setupSchema()` runs GORM AutoMigrate on startup
- Token storage uses SHA-256 hash for access tokens, AES-256-GCM encryption for refresh tokens

**pkg/providers/** - OAuth Provider Integration

- `Provider` interface abstracts OAuth operations (GetAuthorizationURL, ExchangeCodeForToken, GetUserInfo, RefreshToken, GetJWKS)
- `GenericProvider` implements auto-discovery via `.well-known/openid-configuration`
- `ProviderManager` handles multiple providers with concurrent access safety

**pkg/tokens/manager.go - `TokenManager`**

- Generates RS256 JWT access tokens with custom claims
- Validates tokens against JWKS endpoint
- **Important:** Provider's JWKS is fetched and cached, not proxy's own JWKS

### Database Schema Key Points

**Tables:** client_infos, grants, token_datas, authorization_codes, stored_auth_requests

**Critical Relationships:**

- `grants` table stores **encrypted OAuth provider tokens** (not MCP OAuth Proxy tokens)
- One grant per user-client pair (updated on re-authorization)
- `token_datas` stores **issued MCP OAuth Proxy tokens** (access_token as SHA-256 hash, refresh_token encrypted)
- Foreign key: `token_datas.grant_id → grants.id`

**Cleanup:** Expired tokens, auth codes, and auth requests should be cleaned up periodically (methods available but not automated)

## Code Patterns & Conventions

### Interface-Based Dependencies

Handlers define minimal interfaces for their database needs rather than depending on full `Store`:

```go
type TokenStore interface {
    GetClient(clientID string) (*types.ClientInfo, error)
    StoreToken(token *types.TokenData) error
    ValidateAuthCode(code string) (string, string, error)
    // ... only methods this handler needs
}
```

### Error Handling

- Always wrap errors with context: `fmt.Errorf("failed to X: %w", err)`
- OAuth errors use standardized `types.OAuthError` struct with `error` and `error_description`
- HTTP errors sent via `handlerutils.JSON(w, statusCode, errorStruct)`

### Database Operations

- Use GORM methods (`First`, `Create`, `Save`, `Delete`)
- DSN determines database type automatically
- Change logger to `logger.Info` in `db.New()` for debugging SQL queries

### Testing Strategy

- Unit tests use `-short` flag to skip integration tests
- Integration tests require PostgreSQL or use SQLite
- Mock providers for testing external OAuth flows without real providers
- Table-driven tests for multiple scenarios

## Environment Variables

**Required:**

- `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET` - External provider credentials
- `OAUTH_AUTHORIZE_URL` - Provider base URL (e.g., `https://accounts.google.com`)
- `SCOPES_SUPPORTED` - Comma-separated scopes
- `MCP_SERVER_URL` - Backend MCP server endpoint
- `ENCRYPTION_KEY` - Base64-encoded 32-byte AES key (`openssl rand -base64 32`)

**Optional:**

- `DATABASE_DSN` - PostgreSQL connection string (empty = SQLite)
- `PORT` - HTTP server port (default: 8080)

## Common Tasks

### Adding a New OAuth Endpoint

1. Create handler package in `pkg/oauth/newhandler/`
2. Define handler struct and minimal database interface
3. Implement `NewHandler()` and `ServeHTTP()`
4. Add route in `pkg/proxy/proxy.go:SetupRoutes()`
5. Write tests in `newhandler_test.go`

### Adding Database Operations

1. Add method to `Store` in `pkg/db/db.go`
2. If new table needed, add type to `pkg/types/db_types.go` and update `setupSchema()`
3. Write tests in `pkg/db/db_test.go`

### Debugging OAuth Flow

- Set GORM logger to `logger.Info` in `pkg/db/db.go:New()` to see SQL queries
- Add `log.Printf()` statements in handlers to trace flow
- Check `stored_auth_requests` table for pending authorization state
- Verify PKCE: `code_challenge` must match `BASE64URL(SHA256(code_verifier))`

### Working with Providers

- `GenericProvider` fetches `.well-known/openid-configuration` automatically
- Custom providers must implement `Provider` interface
- JWKS is cached in provider manager
- Test providers using `MockProvider` in `pkg/providers/provider_test.go`

## Security Considerations

- **PKCE is mandatory** for public clients to prevent authorization code interception
- Access tokens are **stateless JWTs** (can't be revoked without DB check)
- Refresh tokens are **single-use** and rotated on refresh (implement rotation in token handler)
- OAuth provider tokens are **stored encrypted** in grants table
- Rate limiting is **IP-based** with token bucket algorithm

## Database Migration

Schema changes happen automatically via GORM AutoMigrate. For complex migrations:

1. Add/modify struct fields in `pkg/types/db_types.go`
2. GORM handles column additions on next startup
3. For destructive changes, write manual SQL migration

PostgreSQL required for production multi-instance deployments (shared state). SQLite suitable for development only.

## Workspace Integration

This project is part of the AI workspace. Additional resources:

- **Claude Code commands**: `AI/.claude/commands/` (expert-mode, etc.)
- **Shared agents**: `AI/.claude/agents/` (explore, security-audit, etc.)
- **SuperClaude skills**: `/sc:analyze`, `/sc:test`, `/sc:git`, etc.
- **Serena memories**: `AI/.serena/memories/` (task_completion_checklist, etc.)
- **GitHub Actions**: Workspace-level PR review and issue triage

For session initialization with full context, run `/expert-mode` from the workspace root.
