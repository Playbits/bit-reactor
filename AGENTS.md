# BitReactor — OpenCode Agent Instructions

## Project Role

BitReactor is the **shared platform** powering all Playbit products. It provides auth, RBAC, organizations, projects, and infrastructure. Consuming products include FoodSharePoint, SchoolCare, and PlayCMS.

See `.opencode/context/project/bitreactor.md` for full architecture and ecosystem overview.

## Project Structure

Monorepo with two submodules:

```
bitreactor/
├── backend/   → Go/Gin/GORM API (bitreactor-backend)
└── frontend/  → React/Vite/TanStack Router (bitreactor-frontend)
```

## Quick Start

```bash
# Backend (port :8081)
cd backend && air

# Frontend (port :5173)
cd frontend && yarn dev
```

- DB: PostgreSQL on `localhost:5432`, network `playbit-shared-network`
- Redis: `localhost:6379`
- Start script: `../helpers/start_bitreactor.sh`

## Key Conventions

- Frontend uses **yarn** (not npm)
- Org IDs in URLs are **Hashids-encoded** (see `src/lib/hashid.ts`)
- Frontend routes use TanStack Router v1 with manual route tree (`routeTree.gen.tsx`)
- Backend routes in `routes/router.go`, service layer per feature in `internal/`
- `.env` files exist at `backend/.env` and `frontend/.env`

## Cross-Product Auth

- BitReactor is the **auth provider** for all Playbit products
- Consuming products call BitReactor's `/api/v1/auth/*` endpoints to register, login, refresh, and logout
- JWT access tokens are signed with BitReactor's secret — consuming services validate tokens using the shared secret or future SDK
- Refresh tokens use rotation (revoked on use)
- RBAC is managed via organizations → memberships → projects (owner, admin, member roles)
- Future SDK (Go + TypeScript) will formalize integration for consuming products

## Build & Verify

```bash
cd backend && go build ./...
cd frontend && yarn build
```
