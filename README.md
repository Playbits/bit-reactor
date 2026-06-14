# BitReactor

Shared platform backend and frontend for Playbit products — a modular monorepo providing authentication, organization/project management, team collaboration, and infrastructure tooling.

## Structure

```
bitreactor/
├── backend/    → Go/Gin/GORM API          (bit-reactor-be)
├── frontend/   → React/Vite/TanStack UI   (bit-reactor-fe)
└── ...
```

Both submodules live in their own GitHub repos and are linked here for unified development.

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
