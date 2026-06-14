# Integrating Consuming Products with BitReactor

This guide explains how consuming products (FoodSharePoint, SchoolCare, PlayCMS)
integrate with BitReactor — the shared platform providing auth, RBAC,
organizations, projects, and infrastructure for all Playbit products.

---

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  FoodSharePoint  │     │   SchoolCare    │     │    PlayCMS      │
│  (Next.js/Go)    │     │  (Next.js/Go)   │     │  (Nuxt/Next.js) │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         │         ┌──────────────┴──────────────┐         │
         │         │         BitReactor          │         │
         │         │  ┌──────────────────────┐   │         │
         ├─────────┼──┤ /api/v1/auth/*       │   │         │
         │         │  │ /api/v1/organizations │   │         │
         ├─────────┼──┤ /api/v1/projects     │   │         │
         │         │  │ /api/v1/rbac/*       │   │         │
         │         │  └──────────────────────┘   │         │
         │         └──────────────┬──────────────┘         │
         │                        │                        │
         │              ┌─────────┴─────────┐              │
         │              │   PostgreSQL +    │              │
         │              │      Redis        │              │
         │              └───────────────────┘              │
```

Each consuming product is a separate application with its own frontend and
backend. Products call BitReactor's API directly for auth and cross-cutting
concerns, but manage their own domain-specific data.

---

## Authentication Flow

BitReactor is the **single auth provider** for all Playbit products. Users
register once and can access any product their org has provisioned.

### 1. User Registration

**Endpoint:** `POST /api/v1/auth/register`

```json
// Request
{
  "email": "user@example.com",
  "password": "secure-password",
  "first_name": "Jane",
  "last_name": "Doe"
}

// Response (200)
{
  "data": {
    "user": {
      "id": 1,
      "email": "user@example.com",
      "first_name": "Jane",
      "last_name": "Doe",
      "role": "user",
      "two_factor_enabled": false
    },
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "dGhpcyBpcyBhIHJlZnJl..."
  }
}
```

**Important:** Registration creates a BitReactor user account. The consuming
product should redirect to BitReactor's registration page or proxy the request.
After registration, the user receives JWT tokens that are valid across all
Playbit products.

### 2. Login

**Endpoint:** `POST /api/v1/auth/login`

```json
// Request
{
  "email": "user@example.com",
  "password": "secure-password"
}

// Response (200) — no 2FA
{
  "data": {
    "user": { ... },
    "access_token": "eyJ...",
    "refresh_token": "..."
  }
}

// Response (200) — 2FA challenge required
{
  "data": {
    "requires_two_factor": true,
    "challenge_token": "tmp_challenge_token"
  }
}
```

If the user has 2FA enabled, a `challenge_token` is returned. The consuming
product must then prompt for a TOTP code and complete the challenge.

### 3. Complete 2FA Challenge

**Endpoint:** `POST /api/v1/auth/twofa/challenge`

```json
// Request
{
  "challenge_token": "tmp_challenge_token",
  "code": "123456"
}

// Response (200)
{
  "data": {
    "user": { ... },
    "access_token": "eyJ...",
    "refresh_token": "..."
  }
}
```

### 4. Token Refresh

**Endpoint:** `POST /api/v1/auth/refresh`

```json
// Request
{
  "refresh_token": "current_refresh_token"
}

// Response (200)
{
  "data": {
    "access_token": "eyJ...",
    "refresh_token": "new_refresh_token"
  }
}
```

**Key behaviors:**
- Refresh tokens use **rotation** — each refresh invalidates the previous token
- Refresh tokens expire after 7 days (configurable)
- If a refresh fails (401), the user must log in again
- Tokens are stored by BitReactor and shared across products via the same JWT
  signing secret

### 5. Logout

**Endpoint:** `POST /api/v1/auth/logout`

```json
// Header
Authorization: Bearer <access_token>
```

This revokes the current refresh token. The consuming product should clear
local token storage after a successful logout.

---

## JWT Token Validation

BitReactor signs access tokens with a shared HMAC-SHA256 secret. Consuming
products can validate tokens independently **without** calling BitReactor on
every request.

### Token Structure

```json
// Decoded JWT payload
{
  "sub": 1,
  "email": "user@example.com",
  "role": "user",
  "type": "access",
  "iat": 1680000000,
  "exp": 1680003600
}
```

| Claim  | Description |
|--------|-------------|
| `sub`  | User ID |
| `email` | User email |
| `role`  | Global role (`user` or `admin`) |
| `type`  | Always `"access"` (vs `"refresh"` for refresh tokens) |
| `exp`   | Expiration timestamp (default: 1 hour) |

### Validation in Go

```go
import (
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    Email string `json:"email"`
    Role  string `json:"role"`
    Type  string `json:"type"`
    jwt.RegisteredClaims
}

func ValidateToken(tokenString, secret string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{},
        func(token *jwt.Token) (interface{}, error) {
            return []byte(secret), nil
        },
    )
    if err != nil {
        return nil, err
    }
    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }
    if claims.Type != "access" {
        return nil, errors.New("not an access token")
    }
    return claims, nil
}
```

### Validation in Node.js/TypeScript

```typescript
import jwt from 'jsonwebtoken'

interface Claims {
  sub: number
  email: string
  role: string
  type: string
  iat: number
  exp: number
}

function validateToken(token: string, secret: string): Claims {
  const decoded = jwt.verify(token, secret, { algorithms: ['HS256'] }) as Claims
  if (decoded.type !== 'access') {
    throw new Error('Not an access token')
  }
  return decoded
}
```

**Security note:** The JWT secret must be shared securely between BitReactor
and consuming products (e.g., via environment variable or secrets manager).
The secret is configured in BitReactor's `config.yaml` as `jwt.secret`.

---

## API Patterns

### Base URL

All BitReactor API endpoints are under:

```
https://api.bitreactor.playbit.com/api/v1
```

For local development:

```
http://localhost:8081/api/v1
```

### Authentication Header

All authenticated requests require:

```
Authorization: Bearer <access_token>
```

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable error message"
  }
}
```

Common HTTP status codes:
| Code | Meaning |
|------|---------|
| 200  | Success |
| 400  | Validation error |
| 401  | Unauthenticated (missing/invalid token) |
| 403  | Forbidden (insufficient permissions) |
| 404  | Resource not found |
| 500  | Server error |

### Pagination

List endpoints use query parameters:

```
GET /api/v1/organizations?page=1&per_page=20
```

Response includes pagination metadata:

```json
{
  "data": {
    "organizations": [...],
    "total": 42,
    "page": 1,
    "per_page": 20,
    "total_pages": 3
  }
}
```

### Organization ID Encoding

Organization IDs in frontend URLs are Hashids-encoded. The backend always
receives raw numeric IDs. The encoding is frontend-only, but if your product
needs to construct URLs pointing to BitReactor's frontend, use the same Hashids
salt and minimum-length configuration:

```typescript
// Frontend only — backend receives numeric IDs
import Hashids from 'hashids'

const orgHashids = new Hashids('bitreactor-org', 8)
const projectHashids = new Hashids('bitreactor-project', 8)

encodeOrgId(1)     // → "vX9eL0DV"
decodeOrgId("vX9eL0DV") // → 1
```

---

## Key API Endpoints

### Auth

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Login (may return 2FA challenge) |
| POST | `/auth/refresh` | Refresh tokens |
| POST | `/auth/logout` | Revoke refresh token |
| GET  | `/auth/me` | Get current user |
| POST | `/auth/twofa/setup` | Start 2FA setup |
| POST | `/auth/twofa/verify-setup` | Complete 2FA setup |
| POST | `/auth/twofa/challenge` | Complete 2FA challenge |
| POST | `/auth/twofa/disable` | Disable 2FA |

### Organizations

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/organizations` | List user's orgs |
| POST | `/organizations` | Create org |
| GET | `/organizations/:id` | Get org details |
| PUT | `/organizations/:id` | Update org |
| DELETE | `/organizations/:id` | Delete org |
| GET | `/organizations/:id/members` | List members |
| POST | `/organizations/:id/invites` | Invite member |
| PUT | `/organizations/:id/members/:userId/role` | Change role |
| DELETE | `/organizations/:id/members/:userId` | Remove member |

### Projects

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/organizations/:id/projects` | List org projects |
| POST | `/organizations/:id/projects` | Create project |
| GET | `/projects/:id` | Get project |
| PUT | `/projects/:id` | Update project |
| DELETE | `/projects/:id` | Delete project |

### RBAC

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/organizations/:id/roles` | List roles |
| POST | `/organizations/:id/roles` | Create role |
| PUT | `/roles/:id` | Update role |
| DELETE | `/roles/:id` | Delete role |
| GET | `/projects/:id/roles` | List project roles |
| GET | `/organizations/:id/permissions` | List permissions |
| POST | `/organizations/:id/roles/:roleId/permissions` | Assign permissions |
| DELETE | `/organizations/:id/roles/:roleId/permissions/:permId` | Remove permission |
| GET | `/projects/:id/users` | List project users with roles |
| PUT | `/projects/:id/users/:userId/role` | Set user's project role |

---

## Product-Specific Considerations

### FoodSharePoint
- Uses BitReactor for **user auth and org membership only**
- FoodSharePoint manages its own: meals, menus, orders, deliveries, inventory
- Users are identified by `sub` (user ID) from the JWT, stored in
  FoodSharePoint's own database as a foreign key
- Org-scoped resources use BitReactor's `organization_id` as the scope

### SchoolCare
- Uses BitReactor for **auth + org/project membership + RBAC**
- SchoolCare uses BitReactor's RBAC system for role-based access to school
  resources (students, classes, grades, attendance)
- Custom roles with specific permissions can be created per-school (per-org)

### PlayCMS
- Uses BitReactor for **auth + project membership**
- Each PlayCMS site maps to a BitReactor project
- Content (pages, posts, media) is managed independently by PlayCMS
- PlayCMS uses BitReactor's API keys for server-to-server operations

---

## SDK Roadmap

A future SDK will formalize integration for consuming products:

### Go SDK (planned)
```go
import "github.com/playbit/bitreactor-sdk-go"

client := bitreactor.NewClient(bitreactor.Config{
    BaseURL: "https://api.bitreactor.playbit.com/api/v1",
    Secret:  os.Getenv("BITREACTOR_JWT_SECRET"),
})

// Auth
user, tokens, err := client.Auth.Login(ctx, "email", "password")

// Token validation (cached, no API call)
claims, err := client.ValidateToken(tokenString)

// Organizations
orgs, err := client.Organizations.List(ctx)
```

### TypeScript SDK (planned)
```typescript
import { BitReactorClient } from '@playbit/bitreactor-sdk'

const client = new BitReactorClient({
  baseUrl: 'https://api.bitreactor.playbit.com/api/v1',
  secret: process.env.BITREACTOR_JWT_SECRET!,
})

// Auth
const { user, tokens } = await client.auth.login(email, password)

// Token validation (local, no API call)
const claims = client.validateToken(tokenString)

// Organizations
const orgs = await client.organizations.list()
```

The SDK will handle:
- Token lifecycle management (refresh, retry on 401)
- JWT validation and caching
- Type-safe API responses
- Error handling and rate limiting

---

## Environment Setup for Local Development

Each consuming product's `.env` should include:

```bash
# BitReactor API
BITREACTOR_API_URL=http://localhost:8081/api/v1
BITREACTOR_JWT_SECRET=your-shared-secret

# Infrastructure (shared via playbit-shared-network)
DATABASE_URL=postgres://...
REDIS_URL=redis://localhost:6379
```

Docker services (PostgreSQL, Redis) are shared across all products via the
`playbit-shared-network` Docker network.

For local development against a local BitReactor instance:

```bash
# Start BitReactor (from bitreactor repo)
cd backend && air  # Go API on :8081

# Start consuming product
cd foodsharepoint && yarn dev  # or go run .
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| 401 on every request | Token expired or invalid | Call `/auth/refresh` with the refresh token |
| 403 on resources | User lacks role/permission | Check org membership and RBAC role assignments |
| `challenge_token` on login | User has 2FA enabled | Show OTP prompt and call `/auth/twofa/challenge` |
| Refresh returns 401 | Refresh token expired or rotated | Redirect user to login |
| CORS errors | Missing CORS config | BitReactor must allow the product's origin |
