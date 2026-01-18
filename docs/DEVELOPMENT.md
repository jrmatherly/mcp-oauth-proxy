# Development Guide

Complete guide for developing and contributing to MCP OAuth Proxy.

## Table of Contents

- [Getting Started](#getting-started)
- [Development Environment](#development-environment)
- [Project Structure](#project-structure)
- [Development Workflow](#development-workflow)
- [Testing Strategy](#testing-strategy)
- [Code Style Guide](#code-style-guide)
- [Common Development Tasks](#common-development-tasks)
- [Debugging](#debugging)
- [Contributing](#contributing)

---

## Getting Started

### Prerequisites

**Required:**

- Go 1.25.0 or higher
- Git

**Optional:**

- Docker (for containerized development)
- PostgreSQL 15+ (for integration testing)
- golangci-lint (for code quality)
- Make (build automation)

### Quick Setup

```bash
# Clone the repository
git clone https://github.com/obot-platform/mcp-oauth-proxy.git
cd mcp-oauth-proxy

# Install dependencies
go mod download

# Run tests
make test-short

# Build the application
make build

# Run with SQLite (development mode)
./demo_sqlite.sh
```

---

## Development Environment

### Environment Variables

Create a `.env` file for local development:

```bash
# OAuth Provider Configuration
OAUTH_CLIENT_ID=your-google-client-id
OAUTH_CLIENT_SECRET=your-google-client-secret
OAUTH_AUTHORIZE_URL=https://accounts.google.com
SCOPES_SUPPORTED=openid,email,profile,https://www.googleapis.com/auth/gmail.readonly

# MCP Server
MCP_SERVER_URL=http://localhost:9000/mcp/gmail

# Database (optional, defaults to SQLite)
DATABASE_DSN=postgres://user:pass@localhost:5432/oauth_proxy?sslmode=disable

# Security
ENCRYPTION_KEY=$(openssl rand -base64 32)

# Server (optional)
PORT=8080
HOST=localhost
```

### IDE Setup

#### VS Code

Recommended extensions:

- Go (golang.go)
- Go Test Explorer
- Error Lens
- GitLens

**`.vscode/settings.json`:**

```json
{
  "go.useLanguageServer": true,
  "go.lintTool": "golangci-lint",
  "go.lintOnSave": "package",
  "go.formatTool": "goimports",
  "go.testFlags": ["-v"],
  "editor.formatOnSave": true,
  "go.testTimeout": "5m"
}
```

**`.vscode/launch.json`:**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Server",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}/main.go",
      "envFile": "${workspaceFolder}/.env",
      "args": []
    },
    {
      "name": "Debug Tests",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}",
      "args": ["-v", "-run", "TestName"]
    }
  ]
}
```

---

## Project Structure

```
mcp-oauth-proxy/
├── cmd/                    # CLI commands
│   └── root.go            # Root command setup
├── pkg/                    # Core packages
│   ├── db/                # Database layer
│   │   ├── db.go         # Store implementation
│   │   ├── db_test.go    # Database tests
│   │   └── sqlite_test.go # SQLite-specific tests
│   ├── oauth/             # OAuth 2.1 endpoints
│   │   ├── authorize/    # Authorization endpoint
│   │   ├── callback/     # OAuth callback handler
│   │   ├── register/     # Client registration
│   │   ├── revoke/       # Token revocation
│   │   ├── success/      # Success page
│   │   ├── token/        # Token endpoint
│   │   └── validate/     # Token validation
│   ├── proxy/            # MCP proxy functionality
│   │   ├── proxy.go      # Main proxy handler
│   │   └── proxy_test.go # Proxy tests
│   ├── providers/        # OAuth provider integrations
│   │   ├── provider.go   # Provider interface
│   │   ├── generic.go    # Generic OAuth 2.0 provider
│   │   ├── manager.go    # Provider manager
│   │   └── *_test.go     # Provider tests
│   ├── tokens/           # Token management
│   │   ├── manager.go    # JWT token manager
│   │   ├── apikey.go     # API key authentication
│   │   └── *_test.go     # Token tests
│   ├── types/            # Type definitions
│   │   ├── types.go      # Core types
│   │   ├── db_types.go   # Database models
│   │   ├── oauth_types.go # OAuth structures
│   │   └── apikey_types.go # API key types
│   ├── encryption/       # Encryption utilities
│   ├── handlerutils/     # HTTP handler helpers
│   └── ratelimit/        # Rate limiting
├── docs/                  # Documentation
│   ├── API_REFERENCE.md  # API documentation
│   ├── ARCHITECTURE.md   # Architecture guide
│   ├── DEVELOPMENT.md    # This file
│   └── DATABASE_SCHEMA.md # Database schema
├── main.go               # Application entry point
├── main_test.go          # Integration tests
├── go.mod                # Go module definition
├── Makefile              # Build automation
├── Dockerfile            # Container definition
├── docker-compose.yml    # Docker Compose config
└── README.md             # Project overview
```

### Package Responsibilities

| Package | Responsibility | Key Files |
| --------- | --------------- | ----------- |
| `cmd` | CLI setup and commands | `root.go` |
| `db` | Database operations and schema | `db.go` |
| `oauth/*` | OAuth 2.1 endpoint handlers | `*/handler.go` |
| `proxy` | MCP request proxying | `proxy.go` |
| `providers` | OAuth provider integration | `generic.go`, `manager.go` |
| `tokens` | JWT token operations | `manager.go` |
| `types` | Shared data structures | `types.go` |
| `encryption` | Crypto operations | `encryption.go` |

---

## Development Workflow

### 1. Feature Development

```bash
# Create feature branch
git checkout -b feature/your-feature-name

# Make changes and run tests frequently
make test-short

# Format and vet code
go fmt ./...
go vet ./...

# Run full test suite
make test

# Build to verify compilation
make build
```

### 2. Making Changes

#### Adding a New OAuth Endpoint

1. Create handler in `pkg/oauth/newhandler/`
2. Define interface for database operations
3. Implement `ServeHTTP` method
4. Add route in `pkg/proxy/proxy.go:SetupRoutes`
5. Write tests in `newhandler_test.go`
6. Update API documentation

#### Adding Database Operations

1. Add method to `Store` struct in `pkg/db/db.go`
2. Define database model in `pkg/types/db_types.go`
3. Update schema in `setupSchema()` if needed
4. Write tests in `pkg/db/db_test.go`
5. Update DATABASE_SCHEMA.md documentation

#### Adding a New Provider

1. Implement `Provider` interface in `pkg/providers/`
2. Add auto-discovery support if applicable
3. Write provider tests with mock responses
4. Register provider in `ProviderManager`
5. Update provider documentation

### 3. Testing Workflow

```bash
# Run unit tests only (fast)
make test-short
# or
go test -v -short ./...

# Run all tests including integration
make test
# or
go test -v ./...

# Run tests with race detection
make test-race

# Run tests with coverage
make test-coverage
# View coverage report at coverage.html

# Run specific package tests
go test -v ./pkg/db/
go test -v ./pkg/oauth/token/

# Run specific test
go test -v -run TestTokenEndpoint ./...
```

### 4. Code Quality Checks

```bash
# Format code
make fmt
# or
go fmt ./...

# Vet code
make vet
# or
go vet ./...

# Run linter (if golangci-lint installed)
make lint
# or
golangci-lint run --tests=false

# Run all checks
make ci
```

### 5. Commit and Push

```bash
# Stage changes
git add .

# Commit with conventional commit format
git commit -m "feat: add new OAuth endpoint for token introspection"
# or
git commit -m "fix: handle expired refresh tokens correctly"
# or
git commit -m "docs: update API reference for new endpoint"

# Push to remote
git push origin feature/your-feature-name
```

**Conventional Commit Types:**

- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation changes
- `test:` - Test additions/changes
- `refactor:` - Code refactoring
- `chore:` - Maintenance tasks
- `enhance:` - Enhancement to existing feature
- `perf:` - Performance improvements

---

## Testing Strategy

### Test Categories

#### 1. Unit Tests (`-short` flag)

**Purpose:** Fast tests without external dependencies

**Location:** `*_test.go` files alongside source

**Run:** `make test-short` or `go test -v -short ./...`

**Coverage:**

- Provider interface implementations
- Token validation logic
- Utility functions
- Data structure tests

**Example:**

```go
func TestGenerateCodeChallenge(t *testing.T) {
    verifier := "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
    expected := "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

    result := GenerateCodeChallenge(verifier)
    assert.Equal(t, expected, result)
}
```

#### 2. Integration Tests

**Purpose:** Test database operations and OAuth flows

**Location:** `main_test.go`, `pkg/db/db_test.go`

**Run:** `make test` or `go test -v ./...`

**Setup:** Requires PostgreSQL (use `make setup-test-db`)

**Coverage:**

- Database CRUD operations
- End-to-end OAuth flows
- Token lifecycle
- Client registration

**Example:**

```go
func TestIntegrationOAuthFlow(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    // Setup test database
    store, _ := db.New(os.Getenv("TEST_DATABASE_DSN"))
    defer store.Close()

    // Test authorization flow
    // ...
}
```

#### 3. Mock Provider Tests

**Purpose:** Test external OAuth integration without real providers

**Location:** `pkg/providers/provider_test.go`

**Features:**

- Mock authorization URLs
- Simulated token exchange
- Fake user information
- Error condition testing

**Example:**

```go
func TestMockProviderTokenExchange(t *testing.T) {
    provider := &MockProvider{name: "test"}
    tokenInfo, err := provider.ExchangeCodeForToken(
        ctx, "test_code", "client_id", "secret", "http://redirect"
    )
    assert.NoError(t, err)
    assert.NotEmpty(t, tokenInfo.AccessToken)
}
```

### Test Database Setup

#### PostgreSQL (Docker)

```bash
# Start test database
make setup-test-db

# Run integration tests
export TEST_DATABASE_DSN="postgres://test:test@localhost:5432/oauth_test?sslmode=disable"
make test-integration

# Stop and remove test database
make clean-test-db
```

#### SQLite (Default)

```bash
# No setup required - uses in-memory database
make test-sqlite
```

### Test Coverage Goals

- **Overall:** >80% code coverage
- **Critical paths:** >95% (auth flows, token validation)
- **Database layer:** >90%
- **Handler layer:** >85%

### Writing Tests

**Best Practices:**

1. Use table-driven tests for multiple scenarios
2. Test both success and error paths
3. Use meaningful test names (TestFunctionName_Scenario_ExpectedResult)
4. Clean up resources in `defer` statements
5. Use `testify/assert` for assertions
6. Mock external dependencies

**Example Table-Driven Test:**

```go
func TestValidateAuthCode(t *testing.T) {
    tests := []struct {
        name        string
        code        string
        wantGrantID string
        wantUserID  string
        wantErr     bool
    }{
        {
            name:        "valid code",
            code:        "valid_code_123",
            wantGrantID: "grant_123",
            wantUserID:  "user_123",
            wantErr:     false,
        },
        {
            name:        "invalid code",
            code:        "invalid_code",
            wantGrantID: "",
            wantUserID:  "",
            wantErr:     true,
        },
        {
            name:        "expired code",
            code:        "expired_code",
            wantGrantID: "",
            wantUserID:  "",
            wantErr:     true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            grantID, userID, err := store.ValidateAuthCode(tt.code)
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Equal(t, tt.wantGrantID, grantID)
                assert.Equal(t, tt.wantUserID, userID)
            }
        })
    }
}
```

---

## Code Style Guide

### General Principles

1. **Follow Go idioms** - Use effective Go practices
2. **Keep it simple** - Avoid over-engineering
3. **Write self-documenting code** - Clear names over comments
4. **Prefer composition** - Use interfaces and struct embedding
5. **Handle errors explicitly** - Never ignore errors

### Naming Conventions

**Packages:**

- Lowercase, single word when possible
- Examples: `db`, `proxy`, `oauth`, `tokens`

**Types:**

- PascalCase for exported types
- camelCase for unexported types
- Examples: `Store`, `ClientInfo`, `tokenData`

**Functions/Methods:**

- PascalCase for exported
- camelCase for unexported
- Verb-based names
- Examples: `GetClient`, `StoreToken`, `validateAuthCode`

**Variables:**

- camelCase
- Descriptive names
- Examples: `clientID`, `authRequest`, `encryptedToken`

**Constants:**

- PascalCase
- Examples: `AccessTokenCookieName`, `ModeProxy`

### Code Organization

**Import Order:**

```go
import (
    // Standard library
    "crypto/sha256"
    "encoding/base64"
    "fmt"

    // External packages
    "github.com/gorilla/handlers"
    "gorm.io/gorm"

    // Internal packages
    "github.com/obot-platform/mcp-oauth-proxy/pkg/types"
)
```

**Function Order:**

1. Constructor functions (`New*`)
2. Exported methods (alphabetical)
3. Unexported methods (alphabetical)
4. Helper functions

### Documentation

**Package Documentation:**

```go
// Package oauth provides OAuth 2.1 endpoint handlers.
package oauth
```

**Function Documentation:**

```go
// NewHandler creates a new token endpoint handler with the provided database store.
// The handler implements the OAuth 2.1 token endpoint according to RFC 6749.
func NewHandler(db TokenStore) http.Handler {
    return &Handler{db: db}
}
```

**Struct Documentation:**

```go
// Store represents the database connection and operations for OAuth data persistence.
// It supports both PostgreSQL and SQLite databases through GORM.
type Store struct {
    db     *gorm.DB
    dbType string
}
```

### Error Handling

**Return errors, don't panic:**

```go
// Good
func GetClient(id string) (*ClientInfo, error) {
    var client ClientInfo
    if err := db.First(&client, "client_id = ?", id).Error; err != nil {
        return nil, fmt.Errorf("failed to get client: %w", err)
    }
    return &client, nil
}

// Bad
func GetClient(id string) *ClientInfo {
    var client ClientInfo
    db.First(&client, "client_id = ?", id) // ignores error
    return &client
}
```

**Wrap errors with context:**

```go
if err := store.StoreToken(token); err != nil {
    return fmt.Errorf("failed to store access token: %w", err)
}
```

### Interface Design

**Keep interfaces small:**

```go
// Good - focused interface
type TokenStore interface {
    StoreToken(token *types.TokenData) error
    GetToken(token string) (*types.TokenData, error)
    RevokeToken(token string) error
}

// Bad - too many responsibilities
type Store interface {
    StoreToken(token *types.TokenData) error
    GetToken(token string) (*types.TokenData, error)
    StoreClient(client *types.ClientInfo) error
    GetClient(id string) (*types.ClientInfo, error)
    // ... 20 more methods
}
```

---

## Common Development Tasks

### Running Locally

```bash
# With SQLite (quick start)
./demo_sqlite.sh

# With PostgreSQL
export DATABASE_DSN="postgres://user:pass@localhost:5432/oauth_proxy"
export OAUTH_CLIENT_ID="your-client-id"
export OAUTH_CLIENT_SECRET="your-client-secret"
export OAUTH_AUTHORIZE_URL="https://accounts.google.com"
export SCOPES_SUPPORTED="openid,email,profile"
export MCP_SERVER_URL="http://localhost:9000/mcp/gmail"
export ENCRYPTION_KEY=$(openssl rand -base64 32)

go run main.go
```

### Database Migrations

Schema updates are automatic via GORM AutoMigrate:

```go
// In pkg/db/db.go:setupSchema()
func (s *Store) setupSchema() error {
    return s.db.AutoMigrate(
        &types.ClientInfo{},
        &types.Grant{},
        &types.TokenData{},
        &types.AuthorizationCode{},
        &types.StoredAuthRequest{},
    )
}
```

**Manual migration (if needed):**

```bash
# Connect to PostgreSQL
psql $DATABASE_DSN

# Run SQL migration
ALTER TABLE grants ADD COLUMN new_field VARCHAR(255);
```

### Adding Dependencies

```bash
# Add new dependency
go get github.com/new/package

# Update dependencies
go get -u ./...

# Tidy dependencies
go mod tidy

# Verify dependencies
go mod verify
```

### Building Docker Image

```bash
# Build image
make docker-build
# or
docker build -t mcp-oauth-proxy .

# Run container
make docker-run
# or
docker run -p 8080:8080 --env-file .env mcp-oauth-proxy
```

---

## Debugging

### Enable Debug Logging

```bash
# Set log level to debug
export LOG_LEVEL=debug
go run main.go
```

### Debug Database Queries

```go
// In pkg/db/db.go:New()
gormConfig := &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info), // Change from Silent
}
```

### Debug OAuth Flow

Add logging in handlers:

```go
log.Printf("Authorization request: %+v", authReq)
log.Printf("Provider redirect URL: %s", redirectURL)
```

### Common Issues

#### "Client not found"

- Check `OAUTH_CLIENT_ID` environment variable
- Verify client registered in database
- Run dynamic registration endpoint

#### "Invalid token"

- Check token expiration
- Verify JWKS endpoint accessible
- Check encryption key consistency

#### "Database connection failed"

- Verify `DATABASE_DSN` format
- Check PostgreSQL running: `pg_isready`
- Check SQLite file permissions

---

## Contributing

### Pull Request Process

1. Fork the repository
2. Create feature branch: `git checkout -b feature/amazing-feature`
3. Make changes following code style guide
4. Write/update tests (maintain >80% coverage)
5. Update documentation
6. Run full test suite: `make ci`
7. Commit changes: `git commit -m 'feat: add amazing feature'`
8. Push to branch: `git push origin feature/amazing-feature`
9. Open Pull Request

### PR Checklist

- [ ] Tests pass (`make test`)
- [ ] Code formatted (`go fmt ./...`)
- [ ] Code vetted (`go vet ./...`)
- [ ] Linter passes (`make lint`)
- [ ] Documentation updated
- [ ] Commit messages follow conventional commits
- [ ] No breaking changes (or documented)

### Code Review Guidelines

- Keep PRs focused and small
- Respond to review comments promptly
- Update PR based on feedback
- Squash commits before merging

---

## Cross-References

- **API Documentation:** [API_REFERENCE.md](API_REFERENCE.md)
- **Architecture:** [ARCHITECTURE.md](ARCHITECTURE.md)
- **Database Schema:** [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md)
- **Testing Guide:** [../TESTING.md](../TESTING.md)
- **Main README:** [../README.md](../README.md)
