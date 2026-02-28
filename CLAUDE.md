# Sweep — Cleaning Marketplace

## What This Is

Two-sided marketplace connecting customers with independent cleaners in San Francisco. Three user roles: Customer, Cleaner, Admin.

## Project Structure

This is **NOT a monorepo**. Four separate repos live under this directory:

```
sweep/
├── backend/          # Bun + Elysia API server
├── apps/web/         # Next.js 16 web app
├── apps/mobile/      # Expo 52 mobile app
├── packages/shared/  # @sweep/shared npm package
└── *.md              # Design docs (local only, not pushed)
```

Each repo has its own `node_modules`, git history, and deployment. Docs at the root are local reference only.

## Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Bun + Elysia + Prisma + PostgreSQL (PostGIS) |
| Web | Next.js 16 + React 19 + Tailwind 4 + Clerk + Zustand + React Query |
| Mobile | Expo 52 + React Native 0.76 + expo-router + Zustand + React Query |
| Shared | TypeScript + Zod schemas, published as `@sweep/shared` |
| Auth | Clerk (JWT verified via jose JWKS) |
| Payments | Stripe Connect (cleaner payouts) |

## Key Commands

### Backend (`backend/`)
- `bun run dev` — start dev server with watch (port 3050)
- `bun test` — run all tests
- `bun run db:migrate:dev` — run Prisma migrations
- `bun run db:studio` — open Prisma Studio
- `docker compose up -d` — start Postgres (port 5433) + Redis

### Web (`apps/web/`)
- `npm run dev` — start Next.js dev server
- `npm run build` — production build
- `npm run lint` — ESLint
- `npm run test` — run full test suite

### Mobile (`apps/mobile/`)
- `npx expo start` — start Expo dev server
- `npx expo export --platform ios` — verify iOS build
- `npx expo export --platform android` — verify Android build
- `npm run test` — run full test suite

### Shared (`packages/shared/`)
- `bun run build` — compile TypeScript to dist/
- `bun test` — run tests
- `bun run typecheck` — type check without emit

## Important Conventions

### Database
- Docker Postgres is on host port **5433** (not 5432) to avoid conflicts
- PostGIS extensions must exist before Prisma migrations — handled by `prisma/init-extensions.sql`
- Image: `postgis/postgis:15-3.3-alpine` (ARM warning on Apple Silicon is fine)
- Connection: `postgresql://cleaning:cleaning@localhost:5433/cleaning`

### Backend Patterns
- Routes go in `src/routes/` and are registered via `.use()` in `src/index.ts`
- Domain logic in `src/domain/` (booking, idempotency, etc.)
- Auth middleware in `src/middleware/auth.ts` — verifies Clerk JWTs via JWKS
- Tests co-locate with source: `src/middleware/auth.test.ts`
- Use `bun test` — not Jest, not Vitest

### Web Patterns
- App Router with route groups: `(auth)`, `(customer)`, `(cleaner)`, `(admin)`
- Shared UI components in `src/components/ui/` (Button, Card, Input)
- Layout components in `src/components/shared/` (Header, Footer)
- API client in `src/lib/api/client.ts` (axios with bearer token interceptor)
- State: React Query for server state, Zustand for client state
- Styling: Tailwind 4 utility classes, no CSS modules

### Mobile Patterns
- File-based routing via expo-router with route groups: `(auth)`, `(customer)`, `(cleaner)`
- StyleSheet.create for all styles (no styled-components, no NativeWind)
- Auth store in `lib/stores/authStore.ts` (Zustand + expo-secure-store)
- API client in `lib/api/client.ts`

### Shared Package
- Exports: types, schemas (Zod), constants, contracts
- Publish to npm as `@sweep/shared` before consuming in other repos
- Build with `bun run build` before publishing

## Quality Gates (Required)

- Every new function or behavior change must include corresponding tests in the same PR.
- Validate end-to-end flows for changed behavior (automated e2e when available; otherwise documented manual e2e verification).
- Keep code structured, modular, and DRY across backend and frontends. Reuse components/modules instead of copy-pasting logic.
- Before marking work complete, run the full existing automated test suites for each affected repo (not just newly added or targeted tests); all prior tests must pass.
- Run lint, build, and tests after every change (using each repo's local commands).
- Maintain end-to-end type safety across repos (`backend`, `apps/web`, `apps/mobile`, `packages/shared`), including contract and DTO boundaries.

## Branch and PR Workflow (Required)

- For every code change in any repo, create a new branch and open a new PR to merge into `main` unless explicitly instructed otherwise.
- Never commit or push directly to `main` unless the user explicitly asks for that exception.
- Before merging a PR, run code review using a fresh reviewer sub-agent; if any blocking issue is found, fix it and run review again with a new fresh reviewer sub-agent.
- Repeat that fix and fresh-review cycle until the latest sub-agent PR review is clean, then merge.

## Business Rules
- Platform fee: 8% on booking total (breakdown: 1.5% bonus pool, 2.5% processing, 4% ops)
- Cleaners set own hourly rate: $25–$150
- Booking hold TTL: defined in `src/domain/` constants
- Fee basis points: 800 (PLATFORM_FEE_BPS)

## Design Docs Reference
- `PLAN.md` — implementation roadmap (6 slices)
- `API_SPECIFICATION.md` — full REST API contract
- `DATABASE_SCHEMA.md` — Prisma schema documentation
- `FRONTEND_ARCHITECTURE.md` — page routing, components, state management
- `BACKEND_ARCHITECTURE.md` — server architecture, middleware, domain layer
- `SYSTEM_DESIGN_OVERVIEW.md` — product vision, business model, user flows

## Production Guardrails (Required)

- Require pull requests to `main` with required status checks and required approvals before merge.
- Require Code Owner approval for critical paths (`backend`, `auth`, `payments`, `infra`, and workflow/security configs).
- Treat AI-generated output as draft input only; human review and accountability are mandatory before merge.
- CI on every PR must run compile/build validation, the full automated test suite, and static analysis.
- Security gates are mandatory: CodeQL/code scanning and dependency review must pass.
- Release artifacts must have supply-chain provenance attestation (SLSA-aligned build provenance).
- For AI-related changes, run evals continuously on every change; fail PRs when evals are required but missing.
- Follow secure SDLC baseline controls: NIST SSDF (SP 800-218) and OWASP ASVS 5.0.0.
