# Database Schema Documentation

Complete database schema reference for MCP OAuth Proxy.

## Table of Contents

1. [Database Schema Documentation](#database-schema-documentation)
   1. [Table of Contents](#table-of-contents)
   2. [Overview](#overview)
   3. [Schema Diagram](#schema-diagram)
   4. [Tables](#tables)
      1. [client\_infos](#client_infos)
      2. [grants](#grants)
      3. [token\_datas](#token_datas)
      4. [authorization\_codes](#authorization_codes)
      5. [stored\_auth\_requests](#stored_auth_requests)
   5. [Relationships](#relationships)
      1. [Entity Relationship Diagram](#entity-relationship-diagram)
      2. [Relationship Details](#relationship-details)
   6. [Indexes](#indexes)
      1. [Performance Indexes](#performance-indexes)
   7. [Data Lifecycle](#data-lifecycle)
      1. [Authorization Code Flow Lifecycle](#authorization-code-flow-lifecycle)
      2. [Cleanup Jobs](#cleanup-jobs)
   8. [Migration Guide](#migration-guide)
      1. [Automatic Migrations](#automatic-migrations)
      2. [Manual Migrations](#manual-migrations)
      3. [Database Initialization](#database-initialization)
   9. [Cross-References](#cross-references)

---

## Overview

MCP OAuth Proxy uses GORM for database abstraction and supports:

- **PostgreSQL** - Production deployments
- **SQLite** - Development and testing

**Schema Management:** Automatic migrations via GORM AutoMigrate

**Implementation:** [`pkg/db/db.go`](../pkg/db/db.go)

**Type Definitions:** [`pkg/types/db_types.go`](../pkg/types/db_types.go)

---

## Schema Diagram

```
┌─────────────────────────────────────┐
│         client_infos                 │
│  ┌─────────────────────────────┐    │
│  │ client_id (PK)              │    │
│  │ client_secret               │    │
│  │ client_name                 │    │
│  │ redirect_uris               │    │
│  │ grant_types                 │    │
│  │ response_types              │    │
│  │ scope                       │    │
│  │ token_endpoint_auth_method  │    │
│  │ created_at                  │    │
│  │ updated_at                  │    │
│  └─────────────────────────────┘    │
└──────────────────┬──────────────────┘
                   │
                   │ 1:N
                   ▼
┌─────────────────────────────────────┐
│            grants                    │
│  ┌─────────────────────────────┐    │
│  │ id (PK)                     │    │
│  │ client_id (FK)              │────┘
│  │ user_id                     │
│  │ scope                       │
│  │ encrypted_token             │
│  │ encrypted_refresh_token     │
│  │ token_expires_at            │
│  │ refresh_token_expires_at    │
│  │ email                       │
│  │ name                        │
│  │ created_at                  │
│  │ updated_at                  │
│  └─────────────────────────────┘
         │                │
         │ 1:N            │ 1:N
         ▼                ▼
┌──────────────────┐  ┌──────────────────┐
│ token_datas      │  │authorization_codes│
│ ┌──────────────┐ │  │ ┌──────────────┐ │
│ │ id (PK)      │ │  │ │ id (PK)      │ │
│ │ grant_id (FK)│─┘  │ │ code         │ │
│ │ access_token │    │ │ grant_id (FK)│─┘
│ │ refresh_token│    │ │ user_id      │
│ │ token_type   │    │ │ expires_at   │
│ │ expires_at   │    │ │ created_at   │
│ │ scope        │    │ └──────────────┘
│ │ revoked      │    └──────────────────┘
│ │ client_id    │
│ │ user_id      │
│ │ created_at   │
│ └──────────────┘
└──────────────────┘

┌──────────────────────────────────────┐
│    stored_auth_requests              │
│  ┌────────────────────────────────┐  │
│  │ id (PK)                        │  │
│  │ key (UNIQUE)                   │  │
│  │ data (JSON/TEXT)               │  │
│  │ expires_at                     │  │
│  │ created_at                     │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

---

## Tables

### client_infos

**Purpose:** OAuth 2.1 client registrations (RFC 7591)

**Type Definition:** `types.ClientInfo` in [`pkg/types/db_types.go`](../pkg/types/db_types.go)

**Schema:**

| Column | Type | Constraints | Description |
| -------- | ------ | ------------- | ------------- |
| `client_id` | VARCHAR(255) | PRIMARY KEY | OAuth client identifier (UUID) |
| `client_secret` | VARCHAR(255) | NOT NULL | Client secret (hashed for confidential clients) |
| `client_name` | VARCHAR(255) | | Human-readable client name |
| `redirect_uris` | TEXT | NOT NULL | JSON array of redirect URIs |
| `grant_types` | TEXT | | JSON array of grant types (default: `["authorization_code", "refresh_token"]`) |
| `response_types` | TEXT | | JSON array of response types (default: `["code"]`) |
| `scope` | TEXT | | Space-separated scopes |
| `token_endpoint_auth_method` | VARCHAR(50) | | `client_secret_post`, `client_secret_basic`, or `none` |
| `created_at` | TIMESTAMP | NOT NULL | Record creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**

- Primary key on `client_id`

**Example:**

```sql
INSERT INTO client_infos (
    client_id, client_secret, client_name, redirect_uris,
    grant_types, response_types, scope, token_endpoint_auth_method
) VALUES (
    'f47ac10b-58cc-4372-a567-0e02b2c3d479',
    'hashed_secret',
    'My Application',
    '["http://localhost:3000/callback"]',
    '["authorization_code", "refresh_token"]',
    '["code"]',
    'openid email profile',
    'client_secret_post'
);
```

**Go Struct:**

```go
type ClientInfo struct {
    ClientID                  string    `gorm:"primaryKey;size:255"`
    ClientSecret              string    `gorm:"size:255;not null"`
    ClientName                string    `gorm:"size:255"`
    RedirectURIs              string    `gorm:"type:text;not null"` // JSON array
    GrantTypes                string    `gorm:"type:text"`          // JSON array
    ResponseTypes             string    `gorm:"type:text"`          // JSON array
    Scope                     string    `gorm:"type:text"`
    TokenEndpointAuthMethod   string    `gorm:"size:50"`
    CreatedAt                 time.Time
    UpdatedAt                 time.Time
}
```

---

### grants

**Purpose:** User authorization grants and OAuth provider tokens

**Type Definition:** `types.Grant` in [`pkg/types/db_types.go`](../pkg/types/db_types.go)

**Schema:**

| Column | Type | Constraints | Description |
| -------- | ------ | ------------- | ------------- |
| `id` | VARCHAR(255) | PRIMARY KEY | Grant UUID |
| `client_id` | VARCHAR(255) | FOREIGN KEY | References `client_infos.client_id` |
| `user_id` | VARCHAR(255) | NOT NULL | User ID from OAuth provider |
| `scope` | TEXT | | Space-separated authorized scopes |
| `encrypted_token` | TEXT | | Encrypted OAuth provider access token |
| `encrypted_refresh_token` | TEXT | | Encrypted OAuth provider refresh token |
| `token_expires_at` | TIMESTAMP | | Provider access token expiration |
| `refresh_token_expires_at` | TIMESTAMP | | Provider refresh token expiration |
| `email` | VARCHAR(255) | | User email address |
| `name` | VARCHAR(255) | | User display name |
| `created_at` | TIMESTAMP | NOT NULL | Grant creation timestamp |
| `updated_at` | TIMESTAMP | NOT NULL | Last update timestamp |

**Indexes:**

- Primary key on `id`
- Index on `client_id`
- Index on `user_id`
- Composite index on `(client_id, user_id)`

**Example:**

```sql
INSERT INTO grants (
    id, client_id, user_id, scope, encrypted_token,
    encrypted_refresh_token, token_expires_at, email, name
) VALUES (
    'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    'f47ac10b-58cc-4372-a567-0e02b2c3d479',
    '1234567890',
    'openid email profile',
    'encrypted_access_token_data',
    'encrypted_refresh_token_data',
    '2026-01-15 20:57:45',
    'user@example.com',
    'John Doe'
);
```

**Go Struct:**

```go
type Grant struct {
    ID                       string    `gorm:"primaryKey;size:255"`
    ClientID                 string    `gorm:"index;size:255"`
    UserID                   string    `gorm:"index;size:255;not null"`
    Scope                    string    `gorm:"type:text"`
    EncryptedToken           string    `gorm:"type:text"`
    EncryptedRefreshToken    string    `gorm:"type:text"`
    TokenExpiresAt           time.Time
    RefreshTokenExpiresAt    time.Time
    Email                    string    `gorm:"size:255"`
    Name                     string    `gorm:"size:255"`
    CreatedAt                time.Time
    UpdatedAt                time.Time
}
```

**Notes:**

- `encrypted_token` and `encrypted_refresh_token` use AES-256-GCM encryption
- Stores OAuth provider tokens, not MCP OAuth Proxy tokens
- One grant per user-client pair (updated on re-authorization)

---

### token_datas

**Purpose:** Issued MCP OAuth Proxy access and refresh tokens

**Type Definition:** `types.TokenData` in [`pkg/types/db_types.go`](../pkg/types/db_types.go)

**Schema:**

| Column | Type | Constraints | Description |
| -------- | ------ | ------------- | ------------- |
| `id` | INTEGER | PRIMARY KEY AUTO_INCREMENT | Token record ID |
| `grant_id` | VARCHAR(255) | FOREIGN KEY | References `grants.id` |
| `access_token` | VARCHAR(2048) | UNIQUE NOT NULL | Hashed JWT access token |
| `refresh_token` | VARCHAR(2048) | UNIQUE | Encrypted refresh token |
| `token_type` | VARCHAR(50) | NOT NULL | Always `Bearer` |
| `expires_at` | TIMESTAMP | NOT NULL | Access token expiration |
| `scope` | TEXT | | Space-separated scopes |
| `revoked` | BOOLEAN | DEFAULT FALSE | Revocation status |
| `client_id` | VARCHAR(255) | NOT NULL | OAuth client ID |
| `user_id` | VARCHAR(255) | NOT NULL | User ID |
| `created_at` | TIMESTAMP | NOT NULL | Token issuance timestamp |

**Indexes:**

- Primary key on `id`
- Unique index on `access_token`
- Unique index on `refresh_token`
- Index on `grant_id`
- Index on `expires_at` (for cleanup)
- Index on `revoked`

**Example:**

```sql
INSERT INTO token_datas (
    grant_id, access_token, refresh_token, token_type,
    expires_at, scope, client_id, user_id
) VALUES (
    'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    'sha256_hash_of_jwt_token',
    'encrypted_refresh_token',
    'Bearer',
    '2026-01-15 21:57:45',
    'openid email profile',
    'f47ac10b-58cc-4372-a567-0e02b2c3d479',
    '1234567890'
);
```

**Go Struct:**

```go
type TokenData struct {
    ID           uint      `gorm:"primaryKey;autoIncrement"`
    GrantID      string    `gorm:"index;size:255"`
    AccessToken  string    `gorm:"uniqueIndex;size:2048;not null"`
    RefreshToken string    `gorm:"uniqueIndex;size:2048"`
    TokenType    string    `gorm:"size:50;not null"`
    ExpiresAt    time.Time `gorm:"index;not null"`
    Scope        string    `gorm:"type:text"`
    Revoked      bool      `gorm:"index;default:false"`
    ClientID     string    `gorm:"size:255;not null"`
    UserID       string    `gorm:"size:255;not null"`
    CreatedAt    time.Time
}
```

**Notes:**

- `access_token` stored as SHA-256 hash for validation
- `refresh_token` stored encrypted with AES-256-GCM
- Cleanup job removes expired tokens periodically

---

### authorization_codes

**Purpose:** Temporary authorization codes for OAuth code flow

**Type Definition:** `types.AuthorizationCode` in [`pkg/types/db_types.go`](../pkg/types/db_types.go)

**Schema:**

| Column | Type | Constraints | Description |
| -------- | ------ | ------------- | ------------- |
| `id` | INTEGER | PRIMARY KEY AUTO_INCREMENT | Code record ID |
| `code` | VARCHAR(255) | UNIQUE NOT NULL | Authorization code (random) |
| `grant_id` | VARCHAR(255) | FOREIGN KEY | References `grants.id` |
| `user_id` | VARCHAR(255) | NOT NULL | User ID |
| `expires_at` | TIMESTAMP | NOT NULL | Code expiration (10 minutes) |
| `created_at` | TIMESTAMP | NOT NULL | Code creation timestamp |

**Indexes:**

- Primary key on `id`
- Unique index on `code`
- Index on `grant_id`
- Index on `expires_at` (for cleanup)

**Example:**

```sql
INSERT INTO authorization_codes (
    code, grant_id, user_id, expires_at
) VALUES (
    'random_auth_code_xyz789',
    'a1b2c3d4-e5f6-7890-abcd-ef1234567890',
    '1234567890',
    '2026-01-15 20:07:45'
);
```

**Go Struct:**

```go
type AuthorizationCode struct {
    ID        uint      `gorm:"primaryKey;autoIncrement"`
    Code      string    `gorm:"uniqueIndex;size:255;not null"`
    GrantID   string    `gorm:"index;size:255"`
    UserID    string    `gorm:"size:255;not null"`
    ExpiresAt time.Time `gorm:"index;not null"`
    CreatedAt time.Time
}
```

**Notes:**

- Single-use codes (deleted after exchange)
- Short expiration (10 minutes)
- Automatically cleaned up after expiration

---

### stored_auth_requests

**Purpose:** Temporary storage for OAuth authorization request state

**Type Definition:** `types.StoredAuthRequest` in [`pkg/types/db_types.go`](../pkg/types/db_types.go)

**Schema:**

| Column | Type | Constraints | Description |
| -------- | ------ | ------------- | ------------- |
| `id` | INTEGER | PRIMARY KEY AUTO_INCREMENT | Request record ID |
| `key` | VARCHAR(255) | UNIQUE NOT NULL | Unique request key (UUID) |
| `data` | TEXT | NOT NULL | JSON-encoded request data |
| `expires_at` | TIMESTAMP | NOT NULL | Request expiration (30 minutes) |
| `created_at` | TIMESTAMP | NOT NULL | Request creation timestamp |

**Indexes:**

- Primary key on `id`
- Unique index on `key`
- Index on `expires_at` (for cleanup)

**Example:**

```sql
INSERT INTO stored_auth_requests (
    key, data, expires_at
) VALUES (
    'req_f47ac10b-58cc-4372-a567-0e02b2c3d479',
    '{"client_id":"abc","redirect_uri":"http://...","state":"xyz","code_challenge":"..."}',
    '2026-01-15 20:27:45'
);
```

**Go Struct:**

```go
type StoredAuthRequest struct {
    ID        uint      `gorm:"primaryKey;autoIncrement"`
    Key       string    `gorm:"uniqueIndex;size:255;not null"`
    Data      string    `gorm:"type:text;not null"` // JSON
    ExpiresAt time.Time `gorm:"index;not null"`
    CreatedAt time.Time
}
```

**Notes:**

- Stores PKCE parameters and OAuth state
- Deleted after callback or expiration
- Prevents CSRF attacks via state validation

---

## Relationships

### Entity Relationship Diagram

```
ClientInfo (1) ──< (N) Grant
    │
    └──────────────┐
                   │
Grant (1) ──< (N) TokenData
    │
    └──< (N) AuthorizationCode

StoredAuthRequest (independent)
```

### Relationship Details

**ClientInfo → Grant** (One-to-Many)

- One client can have multiple grants (different users)
- Foreign key: `grants.client_id → client_infos.client_id`
- Cascade: Delete grants when client deleted

**Grant → TokenData** (One-to-Many)

- One grant can have multiple token pairs (token rotation)
- Foreign key: `token_datas.grant_id → grants.id`
- Cascade: Delete tokens when grant deleted

**Grant → AuthorizationCode** (One-to-Many)

- One grant can have multiple codes (re-authorization)
- Foreign key: `authorization_codes.grant_id → grants.id`
- Cascade: Delete codes when grant deleted

---

## Indexes

### Performance Indexes

**Frequently Queried:**

- `client_infos.client_id` - Client lookup (PRIMARY KEY)
- `grants.client_id` - Grant lookup by client
- `grants.user_id` - Grant lookup by user
- `grants(client_id, user_id)` - Composite for unique grant
- `token_datas.access_token` - Token validation (UNIQUE)
- `token_datas.refresh_token` - Refresh lookup (UNIQUE)
- `authorization_codes.code` - Code validation (UNIQUE)
- `stored_auth_requests.key` - Request retrieval (UNIQUE)

**Cleanup Indexes:**

- `token_datas.expires_at` - Expired token cleanup
- `authorization_codes.expires_at` - Expired code cleanup
- `stored_auth_requests.expires_at` - Expired request cleanup
- `token_datas.revoked` - Revoked token queries

---

## Data Lifecycle

### Authorization Code Flow Lifecycle

```
1. Authorization Request
   └─> Store: stored_auth_requests (expires: 30 min)

2. Provider Callback
   ├─> Create/Update: grants
   ├─> Store: authorization_codes (expires: 10 min)
   └─> Delete: stored_auth_requests

3. Token Exchange
   ├─> Validate: authorization_codes
   ├─> Store: token_datas (access_token + refresh_token)
   └─> Delete: authorization_codes

4. Token Usage
   └─> Validate: token_datas.access_token (check revoked, expiration)

5. Token Refresh
   ├─> Validate: token_datas.refresh_token
   ├─> Update: grants (provider token refresh)
   └─> Create: token_datas (new token pair)

6. Token Revocation
   └─> Update: token_datas.revoked = true
```

### Cleanup Jobs

**Expired Tokens:**

```sql
DELETE FROM token_datas
WHERE expires_at < NOW() AND revoked = false;
```

**Expired Authorization Codes:**

```sql
DELETE FROM authorization_codes
WHERE expires_at < NOW();
```

**Expired Auth Requests:**

```sql
DELETE FROM stored_auth_requests
WHERE expires_at < NOW();
```

**Implementation:** [`pkg/db/db.go:CleanupExpiredTokens`](../pkg/db/db.go)

---

## Migration Guide

### Automatic Migrations

GORM AutoMigrate handles schema updates automatically:

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

### Manual Migrations

For complex schema changes, use manual SQL migrations:

**PostgreSQL:**

```sql
-- Add new column
ALTER TABLE grants ADD COLUMN new_field VARCHAR(255);

-- Create index
CREATE INDEX idx_grants_new_field ON grants(new_field);

-- Modify column
ALTER TABLE grants ALTER COLUMN scope TYPE TEXT;
```

**SQLite:**

```sql
-- SQLite doesn't support ALTER COLUMN, use table recreation:
CREATE TABLE grants_new (
    id VARCHAR(255) PRIMARY KEY,
    client_id VARCHAR(255),
    -- ... other columns
    new_field VARCHAR(255)
);

INSERT INTO grants_new SELECT *, NULL FROM grants;
DROP TABLE grants;
ALTER TABLE grants_new RENAME TO grants;
```

### Database Initialization

```bash
# PostgreSQL
createdb oauth_proxy
export DATABASE_DSN="postgres://user:pass@localhost/oauth_proxy?sslmode=disable"
go run main.go  # Auto-creates schema

# SQLite
export DATABASE_DSN=""  # Empty for SQLite
go run main.go  # Creates data/oauth_proxy.db
```

---

## Cross-References

- **Architecture:** [ARCHITECTURE.md](ARCHITECTURE.md) for data flow diagrams
- **API Reference:** [API_REFERENCE.md](API_REFERENCE.md) for endpoint usage
- **Development:** [DEVELOPMENT.md](DEVELOPMENT.md) for setup instructions
- **Type Definitions:** [`pkg/types/db_types.go`](../pkg/types/db_types.go)
- **Database Implementation:** [`pkg/db/db.go`](../pkg/db/db.go)
