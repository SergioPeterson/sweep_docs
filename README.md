# Sweep — Documentation

Design docs and specifications for Sweep, a two-sided marketplace connecting customers with independent cleaners in San Francisco.

## Repos

| Repo | Description |
|------|-------------|
| [sweep_webserver](https://github.com/SergioPeterson/sweep_webserver) | Bun + Elysia API server |
| [sweep_web](https://github.com/SergioPeterson/sweep_web) | Next.js 16 web app |
| [sweep_mobile](https://github.com/SergioPeterson/sweep_mobile) | Expo 52 mobile app |

## Docs

| File | Description |
|------|-------------|
| `SYSTEM_DESIGN_OVERVIEW.md` | Product vision, business model, user flows |
| `API_SPECIFICATION.md` | Full REST API contract |
| `DATABASE_SCHEMA.md` | Prisma schema documentation (22 tables) |
| `BACKEND_ARCHITECTURE.md` | Server architecture, middleware, domain layer |
| `FRONTEND_ARCHITECTURE.md` | Page routing, components, state management |
| `INFRASTRUCTURE_DEPLOYMENT.md` | Deployment and infrastructure design |
| `PLAN.md` | Implementation roadmap (6 slices) |
| `TESTING_STRATEGY.md` | Testing approach and guidelines |
| `ENGINEERING_PLAYBOOK.md` | Engineering conventions and practices |
| `IMPLEMENTATION_GUARDRAILS.md` | Quality gates and guardrails |
| `CLAUDE.md` | AI assistant instructions for the project |
| `AGENTS.md` | Agent configuration and context |

## Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | Bun + Elysia + Prisma + PostgreSQL (PostGIS) |
| Web | Next.js 16 + React 19 + Tailwind 4 + Clerk + Zustand + React Query |
| Mobile | Expo 52 + React Native 0.76 + expo-router + Zustand + React Query |
| Shared | TypeScript + Zod schemas (`@sweep/shared`) |
| Auth | Clerk (JWT verified via jose JWKS) |
| Payments | Stripe Connect (cleaner payouts) |
