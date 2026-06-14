# BitReactor — OpenCode Agent Instructions

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

## Build & Verify

```bash
cd backend && go build ./...
cd frontend && yarn build
```
