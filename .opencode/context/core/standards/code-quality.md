# Code Quality Standards — BitReactor

## Frontend (TypeScript/React)

### Route Conventions (BFF Pattern)
- Use `createFileRoute('/path')({...})` for all route definitions
- Data fetching goes in route `loader` functions, not in component `useEffect`
- Components access loader data via `Route.useLoaderData()`
- Form submissions and pagination state remain in components
- Auth handling stays in `__root.tsx` layout component

### Import Conventions
- `@/` alias for src/ imports (e.g., `@/components/ui/button`)
- `api` from `@/lib/api` for all API calls (includes auth interceptors)

### Component Patterns
- State with `useState` only — avoid custom hooks unless shared across 3+ components
- Axios directly in components for mutations (create, update, delete)
- `useEffect` + `api.get()` for subsequent paginated fetches after initial loader

### TypeScript
- Prefer interfaces over types for object shapes
- Type API response interfaces explicitly (no `any`/`unknown` for response data where avoidable)
- Use `as` casts sparingly — prefer proper type annotations

## Backend (Go)

- Standard Go project layout: `cmd/`, `internal/`, `pkg/`
- Handlers in `internal/<package>/handler.go`
- Services in `internal/<package>/service.go`
- Repositories in `internal/<package>/repo.go`
- DTOs in `internal/<package>/dto.go`
- All models in `internal/models/`
