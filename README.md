# BitReactor

Shared platform backend and frontend for Playbit products — a modular monorepo providing authentication, organization/project management, team collaboration, and infrastructure tooling.

## Structure

```
bitreactor/
├── backend/    → Go/Gin/GORM API          (bit-reactor-be)
│   └── internal/
│       ├── auth/        → JWT, login, register, 2FA/TOTP
│       ├── rbac/        → Roles, permissions, permission matrix
│       ├── twofa/       → TOTP setup, verify, challenge flow
│       ├── org/         → Organizations & memberships
│       ├── project/     → Projects & permission checker
│       ├── apikey/      → API key management (br_ prefix)
│       ├── filestore/   → File uploads (local + S3 per-org)
│       ├── activity/    → Polymorphic activity feed
│       ├── tag/         → Color-coded tags
│       ├── team/        → Teams & team assignments
│       ├── group/       → Membership groups
│       ├── audit/       → Immutable audit logs
│       ├── dashboard/   → Real stats endpoint
│       ├── settings/    → Env vars, service connections, storage
│       └── ...
├── frontend/   → React/Vite/TanStack UI   (bit-reactor-fe)
│   └── src/
│       ├── routes/      → TanStack Router manual routes
│       ├── components/  → shadcn/ui components
│       ├── store/       → Zustand auth state
│       └── lib/         → API client, hashids, utils
└── ...
```

Both submodules live in their own GitHub repos and are linked here for unified development.

## Features

| Category | Feature | Status |
|----------|---------|--------|
| **Auth** | JWT access + refresh tokens, bcrypt, token rotation | ✅ |
| **Auth** | 2FA/TOTP with QR code setup and challenge login | ✅ |
| **Users** | Profile editing, admin CRUD, password changes | ✅ |
| **Orgs** | Multi-tenant organizations with invite flow | ✅ |
| **Projects** | Scoped projects with role hierarchy | ✅ |
| **RBAC** | Custom roles & permissions, permission matrix UI | ✅ |
| **API Keys** | `br_` prefix, SHA-256 hashing, project scoping | ✅ |
| **Files** | Upload, type classification, image dimensions, rename, public URLs | ✅ |
| **Storage** | Local disk + S3 per-org via service connections | ✅ |
| **Activity** | Polymorphic audit trail across all entities | ✅ |
| **Audit Logs** | Immutable INSERT-only compliance trail | ✅ |
| **Tags** | Color-coded tags with filtering | ✅ |
| **Teams** | Team assignments and management | ✅ |
| **Groups** | Membership group assignments | ✅ |
| **Settings** | Env vars, service connections, storage engines | ✅ |
| **Dashboard** | Real stats, activity feed, quick access | ✅ |
| | Tasks & milestones | 🔜 |
| | Webhooks & event system | 🔜 |
| | Cross-product SDK (Go + TypeScript) | 🔜 |
| | Docker/CI deployment | 🔜 |

## Prerequisites

- [Go 1.26+](https://go.dev/dl/)
- [Node.js 20+](https://nodejs.org/)
- [Yarn 4+](https://yarnpkg.com/)
- [PostgreSQL 16+](https://www.postgresql.org/) on `localhost:5432`, network `playbit-shared-network`
- [Redis 7+](https://redis.io/) on `localhost:6379`
- (Optional) [air](https://github.com/air-verse/air) for backend live reload: `go install github.com/air-verse/air@latest`

## Quick Start

```bash
# Clone with submodules
git clone --recurse-submodules git@github.com:Playbits/bit-reactor.git
cd bitreactor

# Backend (port :8081)
cd backend
cp .env.example .env          # or use existing .env
go mod tidy
go run .                      # or: air

# Frontend (port :5173) — separate terminal
cd frontend
yarn install
yarn dev
```

### Environment Variables

Both `backend/.env` and `frontend/.env` contain the required configuration. Key backend variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_URL` | `postgres://...` | PostgreSQL connection string |
| `REDIS_URL` | `localhost:6379` | Redis address |
| `JWT_SECRET` | *(required)* | HS256 signing key |
| `JWT_ACCESS_EXPIRY` | `15m` | Access token TTL |
| `JWT_REFRESH_EXPIRY` | `7d` | Refresh token TTL |
| `STORAGE_PATH` | `./uploads` | Local file storage directory |

## Development

```bash
# Backend — live reload
cd backend && air

# Backend — build check
cd backend && go build ./...

# Frontend — dev server
cd frontend && yarn dev

# Frontend — type check + build
cd frontend && yarn tsc --noEmit && yarn build
```

## URLs

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:8081 |
| API Prefix | `/api/v1` |

## Tech Stack

| Layer | Technology |
|-------|------------|
| **API** | Go 1.26, Gin v1.10, GORM |
| **Database** | PostgreSQL 16 |
| **Cache** | Redis 7 (go-redis/v9) |
| **Auth** | JWT (HS256), bcrypt, Redis-backed refresh tokens |
| **UI** | React 19, TypeScript, Vite 6 |
| **Routing** | TanStack Router v1 |
| **State** | Zustand |
| **UI Kit** | shadcn/ui, TailwindCSS 4 |
| **Package mgr** | Yarn 4 |
