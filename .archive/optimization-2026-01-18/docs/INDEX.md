# MCP OAuth Proxy - Documentation Index

Comprehensive knowledge base and documentation for MCP OAuth Proxy.

## Quick Links

### Getting Started

- [Project Overview](../README.md) - What is MCP OAuth Proxy?
- [Quick Start Guide](../README.md#quick-start) - Get up and running in 5 minutes
- [Environment Setup](DEVELOPMENT.md#environment-variables) - Configuration reference

### Core Documentation

- [API Reference](API_REFERENCE.md) - Complete API endpoint documentation
- [Architecture Guide](ARCHITECTURE.md) - System design and components
- [Development Guide](DEVELOPMENT.md) - Developer workflow and best practices
- [Database Schema](DATABASE_SCHEMA.md) - Data models and relationships
- [Testing Guide](../TESTING.md) - Test suite and strategies

---

## Documentation by Role

### For Developers

**Setting Up:**

1. [Development Environment](DEVELOPMENT.md#development-environment)
2. [Project Structure](DEVELOPMENT.md#project-structure)
3. [Running Locally](DEVELOPMENT.md#running-locally)

**Building Features:**

1. [Development Workflow](DEVELOPMENT.md#development-workflow)
2. [Code Style Guide](DEVELOPMENT.md#code-style-guide)
3. [Testing Strategy](DEVELOPMENT.md#testing-strategy)

**Reference:**

- [Package Documentation](DEVELOPMENT.md#package-responsibilities)
- [Common Tasks](DEVELOPMENT.md#common-development-tasks)
- [Debugging Guide](DEVELOPMENT.md#debugging)

### For API Users

**Getting Started:**

1. [OAuth 2.1 Flow Overview](API_REFERENCE.md#oauth-21-endpoints)
2. [Authentication Methods](API_REFERENCE.md#authentication-methods)
3. [Client Registration](API_REFERENCE.md#registration-endpoint)

**API Reference:**

- [Authorization Endpoint](API_REFERENCE.md#authorization-endpoint)
- [Token Endpoint](API_REFERENCE.md#token-endpoint)
- [MCP Proxy Endpoint](API_REFERENCE.md#mcp-proxy-handler)
- [Error Responses](API_REFERENCE.md#error-response-format)

**Integration:**

- [PKCE Implementation](API_REFERENCE.md#pkce-proof-key-for-code-exchange)
- [Rate Limiting](API_REFERENCE.md#rate-limiting)
- [CORS Support](API_REFERENCE.md#cors-support)

### For Architects

**System Design:**

1. [Architecture Overview](ARCHITECTURE.md#system-overview)
2. [Component Architecture](ARCHITECTURE.md#component-architecture)
3. [Data Flow](ARCHITECTURE.md#data-flow)

**Design Decisions:**

- [Security Architecture](ARCHITECTURE.md#security-architecture)
- [Database Design](DATABASE_SCHEMA.md)
- [Deployment Options](ARCHITECTURE.md#deployment-architecture)

**Scalability:**

- [Horizontal Scaling](ARCHITECTURE.md#multi-instance-deployment-horizontal-scale)
- [Performance Considerations](ARCHITECTURE.md#deployment-architecture)

### For DevOps/SRE

**Deployment:**

1. [Docker Deployment](ARCHITECTURE.md#docker-deployment)
2. [Multi-Instance Setup](ARCHITECTURE.md#multi-instance-deployment-horizontal-scale)
3. [Database Selection](DATABASE_SCHEMA.md#overview)

**Operations:**

- [Health Checks](API_REFERENCE.md#health-check)
- [Monitoring Endpoints](API_REFERENCE.md#health--monitoring)
- [Database Cleanup](DATABASE_SCHEMA.md#cleanup-jobs)

**Troubleshooting:**

- [Common Issues](DEVELOPMENT.md#common-issues)
- [Debug Logging](DEVELOPMENT.md#enable-debug-logging)

---

## Documentation by Topic

### OAuth 2.1 Implementation

**Fundamentals:**

- [OAuth 2.1 Overview](ARCHITECTURE.md#component-architecture)
- [Authorization Code Flow](DATABASE_SCHEMA.md#authorization-code-flow-lifecycle)
- [Refresh Token Flow](ARCHITECTURE.md#refresh-token-flow)

**Endpoints:**

- [Authorization](API_REFERENCE.md#authorization-endpoint) - Initiate OAuth flow
- [Token](API_REFERENCE.md#token-endpoint) - Exchange code for tokens
- [Callback](API_REFERENCE.md#callback-endpoint) - Provider callback
- [Revocation](API_REFERENCE.md#revocation-endpoint) - Revoke tokens
- [Registration](API_REFERENCE.md#registration-endpoint) - Dynamic client registration

**Security:**

- [PKCE Support](ARCHITECTURE.md#pkce-proof-key-for-code-exchange)
- [Token Security](ARCHITECTURE.md#token-security)
- [Client Authentication](ARCHITECTURE.md#client-authentication)

### MCP Integration

**Proxy Functionality:**

- [MCP Proxy Handler](API_REFERENCE.md#mcp-proxy-handler)
- [User Context Headers](../README.md#headers-sent-to-mcp-server)
- [Request Flow](ARCHITECTURE.md#data-flow)

**Implementation:**

- [Proxy Component](ARCHITECTURE.md#1-http-server-layer-pkgproxy)
- [Token Validation](ARCHITECTURE.md#4-token-manager-pkgtokens)
- [Header Injection](API_REFERENCE.md#injected-headers-to-mcp-server)

### Database Management

**Schema:**

- [Tables Overview](DATABASE_SCHEMA.md#tables)
- [Relationships](DATABASE_SCHEMA.md#relationships)
- [Indexes](DATABASE_SCHEMA.md#indexes)

**Operations:**

- [Data Lifecycle](DATABASE_SCHEMA.md#data-lifecycle)
- [Cleanup Jobs](DATABASE_SCHEMA.md#cleanup-jobs)
- [Migrations](DATABASE_SCHEMA.md#migration-guide)

**Tables:**

- [client_infos](DATABASE_SCHEMA.md#client_infos) - OAuth clients
- [grants](DATABASE_SCHEMA.md#grants) - User authorizations
- [token_datas](DATABASE_SCHEMA.md#token_datas) - Issued tokens
- [authorization_codes](DATABASE_SCHEMA.md#authorization_codes) - Auth codes
- [stored_auth_requests](DATABASE_SCHEMA.md#stored_auth_requests) - Pending requests

### Provider Integration

**Core Concepts:**

- [Provider Manager](ARCHITECTURE.md#3-provider-manager-pkgproviders)
- [Auto-Discovery](ARCHITECTURE.md#features)
- [Multi-Provider Support](ARCHITECTURE.md#key-components)

**Supported Providers:**

- Google (`https://accounts.google.com`)
- Microsoft (`https://login.microsoftonline.com`)
- GitHub (`https://github.com/login/oauth/authorize`)
- Custom OAuth 2.0 providers (via auto-discovery)

**Implementation:**

- Provider Interface: [`pkg/providers/provider.go`](../pkg/providers/provider.go)
- Generic Provider: [`pkg/providers/generic.go`](../pkg/providers/generic.go)
- Provider Manager: [`pkg/providers/manager.go`](../pkg/providers/manager.go)

### Testing

**Test Categories:**

- [Unit Tests](DEVELOPMENT.md#1-unit-tests--short-flag)
- [Integration Tests](DEVELOPMENT.md#2-integration-tests)
- [Mock Provider Tests](DEVELOPMENT.md#3-mock-provider-tests)

**Running Tests:**

- [Quick Test Commands](DEVELOPMENT.md#3-testing-workflow)
- [Test Database Setup](DEVELOPMENT.md#test-database-setup)
- [Coverage Goals](DEVELOPMENT.md#test-coverage-goals)

**Writing Tests:**

- [Test Best Practices](DEVELOPMENT.md#writing-tests)
- [Table-Driven Tests](DEVELOPMENT.md#example-table-driven-test)
- [Test Structure](../TESTING.md#test-structure)

---

## Code Reference

### Package Structure

```
pkg/
├── db/              Database operations
├── oauth/           OAuth 2.1 endpoints
│   ├── authorize/   Authorization handler
│   ├── callback/    Callback handler
│   ├── register/    Client registration
│   ├── revoke/      Token revocation
│   ├── success/     Success page
│   ├── token/       Token endpoint
│   └── validate/    Token validation
├── proxy/           MCP proxy handler
├── providers/       OAuth provider integration
├── tokens/          JWT token management
├── types/           Type definitions
├── encryption/      Crypto utilities
├── handlerutils/    HTTP helpers
└── ratelimit/       Rate limiting
```

**See:** [Project Structure](DEVELOPMENT.md#project-structure) for detailed breakdown

### Key Components

| Component | Purpose | Documentation |
| ----------- | --------- | --------------- |
| `OAuthProxy` | Main server | [Architecture](ARCHITECTURE.md#1-http-server-layer-pkgproxy) |
| `Store` | Database layer | [Database Schema](DATABASE_SCHEMA.md) |
| `Provider` | OAuth integration | [Architecture](ARCHITECTURE.md#3-provider-manager-pkgproviders) |
| `TokenManager` | JWT operations | [Architecture](ARCHITECTURE.md#4-token-manager-pkgtokens) |

### Type Definitions

**Core Types:**

- `Config` - Server configuration ([`pkg/types/types.go`](../pkg/types/types.go))
- `ClientInfo` - OAuth client ([`pkg/types/db_types.go`](../pkg/types/db_types.go))
- `Grant` - User authorization ([`pkg/types/db_types.go`](../pkg/types/db_types.go))
- `TokenData` - Issued token ([`pkg/types/db_types.go`](../pkg/types/db_types.go))

**OAuth Types:**

- `AuthRequest` - Authorization parameters ([`pkg/types/oauth_types.go`](../pkg/types/oauth_types.go))
- `OAuthError` - Error responses ([`pkg/types/oauth_types.go`](../pkg/types/oauth_types.go))

---

## Environment Configuration

### Required Variables

| Variable | Description | Example |
| ---------- | ------------- | --------- |
| `OAUTH_CLIENT_ID` | OAuth client ID | `abc123.apps.googleusercontent.com` |
| `OAUTH_CLIENT_SECRET` | OAuth client secret | `GOCSPX-...` |
| `OAUTH_AUTHORIZE_URL` | Provider base URL | `https://accounts.google.com` |
| `SCOPES_SUPPORTED` | Comma-separated scopes | `openid,email,profile` |
| `MCP_SERVER_URL` | Backend MCP server | `http://localhost:9000/mcp/gmail` |
| `ENCRYPTION_KEY` | Base64-encoded AES key | `$(openssl rand -base64 32)` |

### Optional Variables

| Variable | Default | Description |
| ---------- | --------- | ------------- |
| `DATABASE_DSN` | SQLite | PostgreSQL connection string |
| `PORT` | `8080` | HTTP server port |
| `HOST` | `localhost` | HTTP server host |
| `LOG_LEVEL` | `info` | Logging level |

**See:** [Environment Variables](../README.md#environment-variables) for complete reference

---

## Common Workflows

### Setup OAuth Provider

1. [Google Setup](../README.md#google-oauth)
2. [Microsoft Setup](../README.md#microsoft-oauth)
3. [GitHub Setup](../README.md#github-oauth)

### Deploy Application

**Docker:**

```bash
docker run -d -p 8080:8080 \
  -e OAUTH_CLIENT_ID="..." \
  -e OAUTH_CLIENT_SECRET="..." \
  -e OAUTH_AUTHORIZE_URL="..." \
  -e SCOPES_SUPPORTED="..." \
  -e MCP_SERVER_URL="..." \
  -e ENCRYPTION_KEY="..." \
  ghcr.io/obot-platform/mcp-oauth-proxy:latest
```

**Binary:**

```bash
export OAUTH_CLIENT_ID="..."
export OAUTH_CLIENT_SECRET="..."
# ... other variables
./mcp-oauth-proxy
```

**See:** [Quick Start](../README.md#quick-start) for detailed instructions

### Client Integration

1. Register client: [POST /oauth/register](API_REFERENCE.md#registration-endpoint)
2. Initiate authorization: [GET /oauth/authorize](API_REFERENCE.md#authorization-endpoint)
3. Exchange code: [POST /oauth/token](API_REFERENCE.md#token-endpoint)
4. Access MCP: [POST /mcp/*](API_REFERENCE.md#mcp-proxy-handler)

**See:** [Authorization Code Flow](DATABASE_SCHEMA.md#authorization-code-flow-lifecycle)

### Development Tasks

**Run Tests:**

```bash
make test-short  # Unit tests
make test        # All tests
make test-race   # Race detection
```

**Build:**

```bash
make build       # Binary
make docker-build # Container
```

**Quality:**

```bash
make fmt         # Format
make lint        # Lint
make ci          # All checks
```

**See:** [Common Development Tasks](DEVELOPMENT.md#common-development-tasks)

---

## Troubleshooting

### Common Issues

**Authentication Failures:**

- [Client not found](DEVELOPMENT.md#client-not-found)
- [Invalid token](DEVELOPMENT.md#invalid-token)

**Database Issues:**

- [Connection failed](DEVELOPMENT.md#database-connection-failed)
- [Migration errors](DATABASE_SCHEMA.md#migration-guide)

**Provider Issues:**

- OAuth provider configuration
- JWKS endpoint errors
- Token exchange failures

**See:** [Debugging Guide](DEVELOPMENT.md#debugging)

### Getting Help

- **Issues:** [GitHub Issues](https://github.com/obot-platform/mcp-oauth-proxy/issues)
- **Documentation:** This documentation index
- **Testing:** [TESTING.md](../TESTING.md)

---

## Contributing

### Getting Started

1. Fork repository
2. Set up development environment: [Development Setup](DEVELOPMENT.md#getting-started)
3. Read contribution guide: [Contributing](DEVELOPMENT.md#contributing)

### Contribution Checklist

- [ ] Code follows style guide
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] CI passes (`make ci`)
- [ ] Conventional commit format

**See:** [Pull Request Process](DEVELOPMENT.md#pull-request-process)

---

## Changelog & Versioning

- **Releases:** [GitHub Releases](https://github.com/obot-platform/mcp-oauth-proxy/releases)
- **Changelog:** [Release Notes](https://github.com/obot-platform/mcp-oauth-proxy/releases)
- **Versioning:** Semantic Versioning (SemVer)

---

## License

Apache License 2.0 - See [LICENSE](../LICENSE)

---

## Navigation

| Document | Purpose | Audience |
| ---------- | --------- | ---------- |
| [README](../README.md) | Project overview & quick start | Everyone |
| [API_REFERENCE](API_REFERENCE.md) | Complete API documentation | API Users, Developers |
| [ARCHITECTURE](ARCHITECTURE.md) | System design & components | Architects, Senior Devs |
| [DEVELOPMENT](DEVELOPMENT.md) | Development guide | Developers |
| [DATABASE_SCHEMA](DATABASE_SCHEMA.md) | Data models & schema | Developers, DBAs |
| [TESTING](../TESTING.md) | Test suite documentation | Developers, QA |
| **INDEX** (this file) | Documentation navigation | Everyone |

---

**Last Updated:** January 15, 2026

**Documentation Version:** 1.0.0

**Project Version:** See [GitHub Releases](https://github.com/obot-platform/mcp-oauth-proxy/releases)
