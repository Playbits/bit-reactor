# BitReactor Project Guide

## Overview
Shared platform backend (Go/Gin/GORM) + frontend (React/Vite/TanStack Router) that powers all Playbit products. BitReactor is the **central auth, RBAC, and project-management layer** consumed by other Playbit applications.

## Ecosystem — Playbit Product Line
BitReactor is the shared platform consumed by:
- **FoodSharePoint** (`/home/playbit/Playbit/foodSharePoint`) — food sharing platform
- **SchoolCare** (`/home/playbit/Playbit/schoolcare`) — school management
- **PlayCMS** (`/home/playbit/Playbit/playcms_v2`) — content management system

Each product uses BitReactor as its **identity and authorization provider**. A future **SDK** will be built to formalize this integration.

## Repositories
- `bitreactor/` — parent repo with submodules
- `bitreactor/backend/` — Go backend (:8081)
- `bitreactor/frontend/` — React frontend (:5173)

## Backend Tech Stack
- Go 1.26, Gin v1.10, GORM v1.25
- PostgreSQL (Docker: shared-postgres :5432)
- Redis (Docker: shared-redis :6379)
- JWT auth (golang-jwt/jwt/v5, bcrypt)
- godotenv for .env loading
- Air for live reload

## Frontend Tech Stack
- React 19, Vite 6, TypeScript 5.7
- TanStack Router v1.168 (BFF pattern with route loaders)
- TanStack Query v5
- Zustand v5 (auth state)
- shadcn/ui, TailwindCSS 4
- Axios for API calls

## Architecture

### Cross-Product Architecture
```
                   ┌─────────────────────┐
                   │     BitReactor      │
                   │  (Auth + RBAC +     │
                   │   Orgs + Projects)  │
                   └──────┬──┬──┬───────┘
                          │  │  │
              ┌───────────┘  │  └───────────┐
              ▼              ▼              ▼
      FoodSharePoint    SchoolCare       PlayCMS
      (food sharing)    (school mgmt)   (content mgmt)
```

### Current App Architecture
- Frontend dev server proxies `/api` → `localhost:8081` (Vite proxy)
- Route loader pattern (BFF): `createFileRoute` with `loader` functions
- Auth: JWT access + refresh tokens, stored in localStorage, validated on root mount
- No SSR — purely client-side SPA

### Auth & RBAC (Product-Facing)
Each consuming product must:
1. **Register users** — via BitReactor's auth endpoints (register, login, refresh, logout)
2. **Manage sessions** — BitReactor issues JWT access tokens + refresh tokens (with rotation)
3. **Authorize requests** — validate tokens via BitReactor's JWT secret or future SDK
4. **Manage RBAC** — organizations, memberships (owner/admin/member roles), project-level permissions
5. **Rotate tokens** — refresh tokens are revoked on use (rotation pattern)

### Future SDK
A formal SDK (language TBD, likely Go + TypeScript) will:
- Wrap BitReactor API calls for auth, RBAC, org/project CRUD
- Provide server-side token validation middleware for consuming services
- Standardize error handling and API interactions

## Key Decisions
- Port `:8081` (not :8080) for backend
- PostgreSQL only (no SQLite)
- Soft delete for most models (GORM DeletedAt)
- Audit logs are INSERT-only, immutable
- JWT HS256 with token rotation (refresh tokens revoked on use)
- Rate limiter is in-memory (per-IP)

## Route Conventions
- Route files use `createFileRoute('/path')({ component, loader })`
- Loaders fetch initial data from backend API
- Components use `Route.useLoaderData()` for initial data
- `useParams({ from: '/path' })` uses the route path string
- routeTree.gen.tsx is manual (must be kept in sync with route files)
