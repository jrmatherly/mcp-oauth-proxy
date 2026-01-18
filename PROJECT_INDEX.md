# Project Index: MCP OAuth Proxy

**Generated:** January 15, 2026
**Version:** 1.0.0
**Language:** Go 1.25.0
**Type:** OAuth 2.1 Authorization Server + MCP Proxy

---

## 📊 Quick Stats

| Metric | Value |
| -------- | ------- |
| Total Go Files | 32 |
| Source Files | 24 |
| Test Files | 8 |
| Packages | 9 |
| Documentation | 5 major docs (2,900+ lines) |
| Lines of Code | ~3,500 (estimated) |

---

## 📁 Project Structure

```
mcp-oauth-proxy/
├── cmd/                       # CLI commands
│   └── root.go               # Cobra CLI root command
├── pkg/                       # Core packages
│   ├── db/                   # Database layer (GORM)
│   │   ├── db.go            # Store implementation (PostgreSQL/SQLite)
│   │   ├── db_test.go       # Database tests
│   │   └── sqlite_test.go   # SQLite-specific tests
│   ├── oauth/               # OAuth 2.1 endpoint handlers
│   │   ├── authorize/       # Authorization endpoint
│   │   ├── callback/        # OAuth callback handler
│   │   ├── register/        # Dynamic client registration
│   │   ├── revoke/          # Token revocation
│   │   ├── success/         # Success page
│   │   ├── token/           # Token endpoint
│   │   └── validate/        # Token validation
│   ├── proxy/               # MCP proxy functionality
│   │   ├── proxy.go         # Main proxy handler
│   │   └── proxy_test.go    # Proxy tests
│   ├── providers/           # OAuth provider integrations
│   │   ├── provider.go      # Provider interface
│   │   ├── generic.go       # Generic OAuth 2.0 provider
│   │   ├── manager.go       # Provider manager
│   │   └── *_test.go        # Provider tests
│   ├── tokens/              # JWT token management
│   │   ├── manager.go       # Token manager
│   │   ├── apikey.go        # API key authentication
│   │   └── *_test.go        # Token tests
│   ├── types/               # Type definitions
│   │   ├── types.go         # Core types
│   │   ├── db_types.go      # Database models
│   │   ├── oauth_types.go   # OAuth structures
│   │   └── apikey_types.go  # API key types
│   ├── encryption/          # Encryption utilities
│   │   ├── encryption.go    # AES-256-GCM
│   │   └── random.go        # Random generation
│   ├── handlerutils/        # HTTP handler helpers
│   │   └── handlerutils.go  # JSON responses, error formatting
│   └── ratelimit/           # Rate limiting
│       └── ratelimiter.go   # Token bucket algorithm
├── docs/                     # Documentation
│   ├── API_REFERENCE.md     # API endpoint documentation (483 lines)
│   ├── ARCHITECTURE.md      # Architecture guide (581 lines)
│   ├── DATABASE_SCHEMA.md   # Database schema (597 lines)
│   ├── DEVELOPMENT.md       # Development guide (806 lines)
│   └── INDEX.md             # Documentation hub (402 lines)
├── .github/workflows/        # CI/CD pipelines
│   ├── test.yml             # Test workflow
│   ├── build.yml            # Docker build & publish
│   ├── release.yml          # Release automation
│   └── docs.yml             # Documentation generation
├── main.go                   # Application entry point
├── main_test.go             # Integration tests
├── go.mod                   # Go module definition
├── Makefile                 # Build automation
├── Dockerfile               # Container definition
├── docker-compose.yml       # Docker Compose config
└── *.sh                     # Shell scripts (demo, test)
```

---

## 🚀 Entry Points

### Main Entry Point

- **File:** `main.go`
- **Purpose:** Application bootstrap, calls `cmd.Execute()`
- **Package:** `main`

### CLI Command

- **File:** `cmd/root.go`
- **Purpose:** Cobra CLI setup, server initialization
- **Package:** `github.com/obot-platform/mcp-oauth-proxy/cmd`
- **Command:** `mcp-oauth-proxy` (starts HTTP server)

### Testing

- **Integration Tests:** `main_test.go` (all OAuth endpoints)
- **Unit Tests:** `pkg/*/test.go` (per-package tests)
- **Test Scripts:** `run_tests.sh`, `test.sh`, `test_oauth.sh`, `test_sqlite_ci.sh`

### Development

- **Quick Start:** `demo_sqlite.sh` (SQLite demo)
- **Makefile:** Common tasks (test, build, lint, docker)

---

## 📦 Core Packages

### pkg/proxy - HTTP Server & MCP Proxy

**File:** `pkg/proxy/proxy.go`

**Exports:**

- `OAuthProxy` struct - Main server
- `NewOAuthProxy()` - Server constructor
- `Start()` - Start HTTP server
- `SetupRoutes()` - Configure routes
- `GetHandler()` - Get HTTP handler

**Purpose:** HTTP server, CORS, rate limiting, MCP request proxying

**Key Constants:**

- `ModeProxy` - Proxy mode
- `ModeForwardAuth` - Forward auth mode
- `ModeMiddleware` - Middleware mode

---

### pkg/oauth/* - OAuth 2.1 Endpoints

#### oauth/authorize

**File:** `pkg/oauth/authorize/authorize.go`

**Exports:**

- `Handler` struct
- `NewHandler()` - Authorization endpoint handler
- `AuthorizationStore` interface

**Purpose:** OAuth 2.1 authorization endpoint (`/oauth/authorize`)

**Features:** PKCE support, client validation, provider redirect

---

#### oauth/token

**File:** `pkg/oauth/token/token.go`

**Exports:**

- `Handler` struct
- `NewHandler()` - Token endpoint handler
- `TokenStore` interface

**Purpose:** Token endpoint (`/oauth/token`)

**Grant Types:**

- Authorization code grant (with PKCE)
- Refresh token grant

---

#### oauth/callback

**File:** `pkg/oauth/callback/callback.go`

**Exports:**

- `Handler` struct
- `NewHandler()` - Callback handler
- `Store` interface
- `MCPUIManager` interface

**Purpose:** OAuth provider callback (`/oauth/callback`)

**Flow:** Exchange code → fetch user info → create grant → redirect

---

#### oauth/revoke

**File:** `pkg/oauth/revoke/revoke.go`

**Exports:**

- `Handler` struct
- `NewHandler()` - Revocation endpoint handler
- `Store` interface

**Purpose:** Token revocation (`/oauth/revoke`)

---

#### oauth/register

**File:** `pkg/oauth/register/register.go`

**Exports:**

- `Handler` struct
- `NewHandler()` - Registration endpoint handler
- `ClientStore` interface

**Purpose:** Dynamic client registration (`/oauth/register`, RFC 7591)

---

#### oauth/validate

**File:** `pkg/oauth/validate/validatetoken.go`

**Purpose:** Token validation utilities

---

#### oauth/success

**File:** `pkg/oauth/success/success.go`

**Purpose:** Success page handler

---

### pkg/db - Database Layer

**File:** `pkg/db/db.go`

**Exports:**

- `Store` struct - Database operations
- `New()` - Create database connection

**Methods:**

- Client: `GetClient()`, `StoreClient()`
- Grant: `StoreGrant()`, `UpdateGrant()`, `GetGrant()`
- Token: `StoreToken()`, `GetToken()`, `RevokeToken()`
- Auth Code: `StoreAuthCode()`, `ValidateAuthCode()`, `DeleteAuthCode()`
- Cleanup: `CleanupExpiredTokens()`, `CleanupExpiredAuthRequests()`

**Purpose:** Database abstraction (PostgreSQL/SQLite via GORM)

**Supports:**

- PostgreSQL (production)
- SQLite (development)

---

### pkg/providers - OAuth Provider Integration

**Files:**

- `provider.go` - Provider interface
- `generic.go` - Generic OAuth 2.0 provider
- `manager.go` - Provider manager

**Exports:**

- `Provider` interface - OAuth provider operations
- `GenericProvider` - Auto-discovery implementation
- `ProviderManager` - Multi-provider management

**Methods:**

- `GetAuthorizationURL()` - Generate auth URL
- `ExchangeCodeForToken()` - Token exchange
- `GetUserInfo()` - Fetch user information
- `RefreshToken()` - Refresh access token
- `GetJWKS()` - Get JSON Web Key Set

**Purpose:** OAuth 2.0 provider integration with auto-discovery

**Supported Providers:**

- Google, Microsoft, GitHub
- Any OAuth 2.0 provider (via auto-discovery)

---

### pkg/tokens - JWT Token Management

**Files:**

- `manager.go` - JWT operations
- `apikey.go` - API key authentication

**Exports:**

- `TokenManager` struct - JWT generation and validation
- `NewTokenManager()` - Token manager constructor

**Methods:**

- `GenerateToken()` - Generate JWT access token
- `ValidateToken()` - Validate JWT signature
- `GenerateAPIKey()` - Generate API key
- `ValidateAPIKey()` - Validate API key

**Purpose:** JWT (RS256) token operations and API key auth

**Features:**

- RS256 signing
- JWKS endpoint
- Custom claims
- Token validation

---

### pkg/types - Type Definitions

**Files:**

- `types.go` - Core types
- `db_types.go` - Database models
- `oauth_types.go` - OAuth structures
- `apikey_types.go` - API key types

**Key Types:**

- `Config` - Server configuration
- `ClientInfo` - OAuth client metadata
- `Grant` - User authorization grant
- `TokenData` - Issued token information
- `AuthRequest` - Authorization parameters
- `OAuthError` - Error responses
- `AuthorizationCode` - Authorization code
- `StoredAuthRequest` - Pending auth request

**Purpose:** Shared data structures across packages

---

### pkg/encryption - Encryption Utilities

**Files:**

- `encryption.go` - AES encryption
- `random.go` - Random generation

**Exports:**

- `Encrypt()` - AES-256-GCM encryption
- `Decrypt()` - AES-256-GCM decryption
- `GenerateRandomString()` - Secure random strings
- `GenerateCodeChallenge()` - PKCE code challenge

**Purpose:** Cryptographic operations for token encryption and PKCE

---

### pkg/handlerutils - HTTP Utilities

**File:** `handlerutils.go`

**Exports:**

- `JSON()` - JSON response helper
- `Error()` - Error response helper

**Purpose:** HTTP handler utilities and response formatting

---

### pkg/ratelimit - Rate Limiting

**File:** `ratelimiter.go`

**Exports:**

- `RateLimiter` struct
- `NewRateLimiter()` - Rate limiter constructor

**Purpose:** IP-based rate limiting with token bucket algorithm

---

## 🔧 Configuration Files

### Go Module

- **File:** `go.mod`
- **Module:** `github.com/obot-platform/mcp-oauth-proxy`
- **Go Version:** 1.25.0

### Build Automation

- **File:** `Makefile`
- **Targets:** test, build, lint, docker-build, demo-sqlite, ci

### Docker

- **Files:** `Dockerfile`, `docker-compose.yml`
- **Image:** `ghcr.io/obot-platform/mcp-oauth-proxy:latest`

### CI/CD

- **Directory:** `.github/workflows/`
- **Workflows:**
  - `test.yml` - Run tests on push/PR
  - `build.yml` - Build and publish Docker image
  - `release.yml` - Create GitHub releases
  - `docs.yml` - Generate documentation

---

## 📚 Documentation

### Main Documentation (docs/)

1. **API_REFERENCE.md** (483 lines)
   - Complete API endpoint reference
   - OAuth 2.1 endpoints (9 endpoints)
   - Request/response formats, examples
   - Authentication methods, PKCE, rate limiting

2. **ARCHITECTURE.md** (581 lines)
   - System architecture and design
   - Component documentation (7 components)
   - Data flow diagrams
   - Security architecture, deployment options

3. **DATABASE_SCHEMA.md** (597 lines)
   - Complete database schema (5 tables)
   - Entity relationships, indexes
   - Data lifecycle and cleanup
   - Migration guide

4. **DEVELOPMENT.md** (806 lines)
   - Development environment setup
   - Project structure, workflows
   - Testing strategy, code style guide
   - Common tasks, debugging, contributing

5. **INDEX.md** (402 lines)
   - Documentation navigation hub
   - Role-based and topic-based navigation
   - Quick links and workflows

### Root Documentation

- **README.md** - Project overview, quick start
- **TESTING.md** - Test suite documentation
- **DOCUMENTATION_SUMMARY.md** - Documentation overview

### Serena Memories (.serena/memories/)

- `project_overview.md` - Project purpose and architecture
- `tech_stack.md` - Go version and dependencies
- `code_style_conventions.md` - Coding standards
- `suggested_commands.md` - Common commands
- `task_completion_checklist.md` - Task workflow
- `codebase_structure.md` - Directory structure

---

## 🧪 Test Coverage

### Test Files

- **Integration Tests:** `main_test.go`
- **Database Tests:** `pkg/db/db_test.go`, `pkg/db/sqlite_test.go`
- **Provider Tests:** `pkg/providers/*_test.go`
- **Token Tests:** `pkg/tokens/*_test.go`
- **Proxy Tests:** `pkg/proxy/proxy_test.go`

### Test Categories

1. **Unit Tests** (`-short` flag)
   - Provider interface implementations
   - Token validation logic
   - Utility functions

2. **Integration Tests**
   - Database operations (CRUD)
   - OAuth flow integration
   - Token lifecycle

3. **Mock Provider Tests**
   - Authorization URL generation
   - Token exchange simulation
   - User info retrieval

### Test Commands

```bash
make test-short      # Unit tests only
make test            # All tests
make test-race       # Race detection
make test-coverage   # Coverage report
make test-sqlite     # SQLite tests
make test-integration # PostgreSQL integration tests
```

### Coverage Goals

- Overall: >80%
- Critical paths: >95%
- Database layer: >90%
- Handler layer: >85%

---

## 🔗 Key Dependencies

### Core Dependencies

| Dependency | Version | Purpose |
| ------------ | --------- | --------- |
| `github.com/spf13/cobra` | v1.7.0 | CLI framework |
| `gorm.io/gorm` | v1.30.1 | ORM framework |
| `gorm.io/driver/postgres` | v1.6.0 | PostgreSQL driver |
| `github.com/glebarez/sqlite` | v1.11.0 | SQLite driver (CGO-free) |

### OAuth & Security

| Dependency | Version | Purpose |
| ------------ | --------- | --------- |
| `golang.org/x/oauth2` | v0.30.0 | OAuth 2.0 client |
| `github.com/golang-jwt/jwt/v5` | v5.3.0 | JWT operations |
| `github.com/MicahParks/keyfunc/v3` | v3.7.0 | JWT key functions |

### HTTP & Middleware

| Dependency | Version | Purpose |
| ------------ | --------- | --------- |
| `github.com/gorilla/handlers` | v1.5.2 | HTTP middleware (CORS) |

### Testing

| Dependency | Version | Purpose |
| ------------ | --------- | --------- |
| `github.com/stretchr/testify` | v1.10.0 | Testing assertions |

---

## 📝 Quick Start

### 1. Prerequisites

```bash
# Required
go version  # Go 1.25.0+

# Optional
docker --version        # For containerized deployment
psql --version         # For PostgreSQL (production)
golangci-lint --version # For linting
```

### 2. Setup

```bash
# Clone repository
git clone https://github.com/obot-platform/mcp-oauth-proxy.git
cd mcp-oauth-proxy

# Install dependencies
go mod download

# Run tests
make test-short

# Build
make build
```

### 3. Run (SQLite Demo)

```bash
# Quick demo with SQLite
./demo_sqlite.sh

# Or manually
export OAUTH_CLIENT_ID="your-client-id"
export OAUTH_CLIENT_SECRET="your-client-secret"
export OAUTH_AUTHORIZE_URL="https://accounts.google.com"
export SCOPES_SUPPORTED="openid,email,profile"
export MCP_SERVER_URL="http://localhost:9000/mcp/gmail"
export ENCRYPTION_KEY=$(openssl rand -base64 32)

go run main.go
```

### 4. Test

```bash
# Unit tests (fast)
make test-short

# All tests
make test

# With coverage
make test-coverage

# Integration tests (requires PostgreSQL)
make setup-test-db
make test-integration
```

### 5. Development Workflow

```bash
# Format code
make fmt

# Run linter
make lint

# Run all checks
make ci

# Build Docker image
make docker-build
```

---

## 🎯 API Endpoints

### OAuth 2.1 Endpoints

- `GET|POST /oauth/authorize` - Authorization endpoint
- `POST /oauth/token` - Token endpoint (auth code, refresh)
- `GET /oauth/callback` - OAuth provider callback
- `POST /oauth/revoke` - Token revocation
- `POST /oauth/register` - Dynamic client registration

### Metadata Endpoints

- `GET /.well-known/oauth-authorization-server` - OAuth metadata (RFC 8414)
- `GET /.well-known/oauth-protected-resource` - Resource metadata (RFC 8705)

### MCP Proxy

- `POST /mcp/*` - MCP request proxy (with token validation)

### Health & Monitoring

- `GET /health` - Health check

---

## 🔐 Environment Variables

### Required

- `OAUTH_CLIENT_ID` - OAuth client ID
- `OAUTH_CLIENT_SECRET` - OAuth client secret
- `OAUTH_AUTHORIZE_URL` - Provider base URL
- `SCOPES_SUPPORTED` - Comma-separated scopes
- `MCP_SERVER_URL` - Backend MCP server URL
- `ENCRYPTION_KEY` - Base64-encoded 32-byte AES key

### Optional

- `DATABASE_DSN` - PostgreSQL connection (defaults to SQLite)
- `PORT` - HTTP server port (default: 8080)
- `HOST` - HTTP server host (default: localhost)
- `LOG_LEVEL` - Logging level (default: info)

---

## 💡 Common Commands

```bash
# Development
make test-short        # Run unit tests
make test              # Run all tests
make build             # Build binary
make demo-sqlite       # Run SQLite demo

# Code Quality
make fmt               # Format code
make vet               # Vet code
make lint              # Run linter
make ci                # Run all checks

# Docker
make docker-build      # Build image
make docker-run        # Run container

# Database
make setup-test-db     # Start PostgreSQL test DB
make clean-test-db     # Stop test DB
```

---

## 📊 Token Efficiency

### Index Benefits

- **Index Size:** ~5KB (this file)
- **Full Codebase:** ~58,000 tokens (all files)
- **Savings:** 94% token reduction
- **ROI:** Immediate (1 session break-even)

### Usage Pattern

- **Initial Setup:** Read full codebase (one-time)
- **Subsequent Sessions:** Read `PROJECT_INDEX.md` (3,000 tokens)
- **Deep Dive:** Use index to locate specific files/packages

---

## 🔍 Finding Code

### By Functionality

- **OAuth Authorization:** `pkg/oauth/authorize/`
- **Token Management:** `pkg/oauth/token/`, `pkg/tokens/`
- **Database:** `pkg/db/`
- **Provider Integration:** `pkg/providers/`
- **MCP Proxy:** `pkg/proxy/`

### By Type

- **Handlers:** `pkg/oauth/*/handler.go`
- **Interfaces:** `pkg/*/interface.go` or in handler files
- **Types:** `pkg/types/*.go`
- **Tests:** `*_test.go`

### By Feature

- **PKCE:** `pkg/db/db.go:GenerateCodeChallenge`, `pkg/oauth/authorize/`
- **JWT:** `pkg/tokens/manager.go`
- **Encryption:** `pkg/encryption/`
- **Rate Limiting:** `pkg/ratelimit/`

---

## 📌 Important Notes

### Database

- Auto-migration on startup (GORM AutoMigrate)
- SQLite for development (data/oauth_proxy.db)
- PostgreSQL for production

### Security

- Access tokens: RS256 JWT (1 hour)
- Refresh tokens: AES-256-GCM encrypted (90 days)
- PKCE required for public clients
- Rate limiting: 100 req/min per IP

### Provider Support

- Google, Microsoft, GitHub (tested)
- Any OAuth 2.0 provider (via auto-discovery)
- JWKS auto-fetching and caching

---

## 🔄 Recent Changes

See [GitHub Releases](https://github.com/obot-platform/mcp-oauth-proxy/releases) for detailed changelog.

**Recent Commits:**

- `3745d9b` - feat: support API key authentication
- `05acb93` - enhance: add support for UI login
- `62d90ee` - enhance: add ability to specify trusted external JWKS
- `6c42ff3` - fix: include path in resource_metadata
- `5515011` - chore: build image on PRs

---

## 📖 Next Steps

### For New Developers

1. Read [README.md](README.md)
2. Follow [DEVELOPMENT.md - Getting Started](docs/DEVELOPMENT.md#getting-started)
3. Run `make test-short` and `make build`
4. Explore `pkg/` packages by functionality

### For API Users

1. Review [API_REFERENCE.md](docs/API_REFERENCE.md)
2. Set up OAuth provider credentials
3. Test with `demo_sqlite.sh`
4. Integrate with your MCP server

### For Contributors

1. Read [DEVELOPMENT.md - Contributing](docs/DEVELOPMENT.md#contributing)
2. Fork repository and create feature branch
3. Follow code style guide and add tests
4. Submit PR with documentation updates

---

**Index Version:** 1.0.0
**Last Updated:** January 15, 2026
**Maintained By:** MCP OAuth Proxy Team
