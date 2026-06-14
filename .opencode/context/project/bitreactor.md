# BitReactor Project Guide

## Overview
Shared platform backend (Go/Gin/GORM) + frontend (React/Vite/TanStack Router) supporting multiple Playbit products.

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
- Frontend dev server proxies `/api` → `localhost:8081` (Vite proxy)
- Route loader pattern (BFF): `createFileRoute` with `loader` functions
- Auth: JWT access + refresh tokens, stored in localStorage, validated on root mount
- No SSR — purely client-side SPA

## Key Decisions
- Port `:8081` (not :8080) for backend
- PostgreSQL only (no SQLite)
- Soft delete for most models (GORM DeletedAt)
- Audit logs are INSERT-only, immutable

## Route Conventions
- Route files use `createFileRoute('/path')({ component, loader })`
- Loaders fetch initial data from backend API
- Components use `Route.useLoaderData()` for initial data
- `useParams({ from: '/path' })` uses the route path string
- routeTree.gen.tsx is manual (must be kept in sync with route files)
