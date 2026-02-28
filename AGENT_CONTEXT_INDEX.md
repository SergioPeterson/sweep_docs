# Sweep Agent Context Index

Purpose: give an agent one place to locate code and docs quickly.

Scope: first-party project files only. Generated/vendor folders are excluded from this map.

---

## Ignore These Paths During Analysis

| Path Pattern | Why Ignore |
|---|---|
| `**/node_modules/**` | Third-party dependencies |
| `apps/web/.next/**` | Next.js generated build/dev output |
| `apps/web/.git/**` | Git internals, not app logic |
| `apps/mobile/.expo/**` | Expo generated metadata |

---

## Top-Level Documentation Map

| Topic / Question | Primary Markdown | Secondary Markdown | Notes |
|---|---|---|---|
| System overview and product model | `SYSTEM_DESIGN_OVERVIEW.md` | `PLAN.md` | High-level architecture and goals |
| API routes, payloads, errors | `API_SPECIFICATION.md` | `BACKEND_ARCHITECTURE.md` | Contract-first reference |
| Backend module design and flows | `BACKEND_ARCHITECTURE.md` | `DATABASE_SCHEMA.md` | Includes auth/payments/search design intent |
| Database tables, indexes, constraints | `DATABASE_SCHEMA.md` | `backend/prisma/schema.prisma` | SQL design + Prisma implementation snapshot |
| Web frontend architecture direction | `FRONTEND_ARCHITECTURE.md` | `apps/web/src/**` | Canonical frontend design guidance |
| Infra, AWS, deployment model | `INFRASTRUCTURE_DEPLOYMENT.md` | `backend/DOCKER.md` | Cloud topology + local container workflow |
| Implementation sequencing | `PLAN.md` | `IMPLEMENTATION_GUARDRAILS.md` | Slice roadmap + cross-cutting guardrails |
| Security, reliability, deployment guardrails | `IMPLEMENTATION_GUARDRAILS.md` | `INFRASTRUCTURE_DEPLOYMENT.md`, `BACKEND_ARCHITECTURE.md` | Current “what to implement next” checklist |

---

## Codebase Map (By Workspace)

| Area | Key Paths | What Lives Here |
|---|---|---|
| Backend API runtime | `backend/src/index.ts` | Elysia server entry, `/health`, `/api/v1/status` |
| Backend DB client | `backend/src/lib/prisma.ts` | Prisma singleton setup |
| Backend schema | `backend/prisma/schema.prisma` | Prisma enums/models (`User`, `Cleaner`, `Booking`) |
| Backend container/local ops | `backend/Dockerfile`, `backend/docker-compose.yml`, `backend/DOCKER.md`, `backend/.env.example` | Container build/run config and local env conventions |
| Web app routes | `apps/web/src/app/**/page.tsx` | App Router pages for auth/customer/cleaner flows |
| Web API access layer | `apps/web/src/lib/api/client.ts`, `apps/web/src/lib/api/cleaners.ts`, `apps/web/src/lib/api/bookings.ts` | Axios client + endpoint wrappers |
| Web data hooks | `apps/web/src/lib/hooks/useAuth.ts`, `apps/web/src/lib/hooks/useBooking.ts`, `apps/web/src/lib/hooks/useGeolocation.ts` | Clerk user hook, React Query hooks, geolocation |
| Web local state | `apps/web/src/lib/stores/searchStore.ts`, `apps/web/src/lib/stores/bookingStore.ts` | Zustand search/booking draft state |
| Web UI components | `apps/web/src/components/**` | Shared UI primitives + search + booking components |
| Web app shell/config | `apps/web/src/app/layout.tsx`, `apps/web/src/app/providers.tsx`, `apps/web/next.config.ts`, `apps/web/eslint.config.mjs`, `apps/web/tsconfig.json` | Root layout, query provider, framework/tooling config |
| Mobile app routes | `apps/mobile/app/**` | Expo Router stacks/tabs and screens |
| Mobile API/auth state | `apps/mobile/lib/api/client.ts`, `apps/mobile/lib/stores/authStore.ts` | Axios + SecureStore token/auth state |
| Mobile app config | `apps/mobile/app.json`, `apps/mobile/tsconfig.json` | Expo app metadata and TS settings |
| Shared contracts package | `packages/shared/src/types/**`, `packages/shared/src/schemas/validation.ts`, `packages/shared/src/constants/index.ts` | Reusable types, zod schemas, constants |

---

## Feature-to-Source Lookup

| Feature / Concern | Start With Docs | Then Read Code | Search Hints (`rg`) |
|---|---|---|---|
| Authentication (Clerk vs platform role) | `API_SPECIFICATION.md`, `BACKEND_ARCHITECTURE.md`, `IMPLEMENTATION_GUARDRAILS.md` | `apps/web/src/lib/hooks/useAuth.ts`, `apps/web/src/lib/api/client.ts`, `backend/prisma/schema.prisma`, `packages/shared/src/schemas/validation.ts` | `Clerk|auth|Bearer|external_auth_id|role` |
| Cleaner search (filters, map, ranking) | `API_SPECIFICATION.md`, `BACKEND_ARCHITECTURE.md`, `DATABASE_SCHEMA.md` | `apps/web/src/app/(customer)/search/page.tsx`, `apps/web/src/components/search/*`, `apps/web/src/lib/api/cleaners.ts`, `apps/web/src/lib/stores/searchStore.ts` | `search/cleaners|minRating|distance|rating|Mapbox|score` |
| Booking creation + holds | `API_SPECIFICATION.md`, `DATABASE_SCHEMA.md`, `BACKEND_ARCHITECTURE.md` | `apps/web/src/lib/api/bookings.ts`, `apps/web/src/lib/stores/bookingStore.ts`, `packages/shared/src/types/booking.ts`, `packages/shared/src/schemas/validation.ts`, `backend/prisma/schema.prisma` | `booking|hold|idempotency|slot|address|payment_intent` |
| Payments + Stripe webhook | `API_SPECIFICATION.md`, `BACKEND_ARCHITECTURE.md`, `IMPLEMENTATION_GUARDRAILS.md` | `apps/web/src/lib/api/bookings.ts`, `backend/prisma/schema.prisma` | `Stripe|webhook|payment_intent|payout|refund|idempot` |
| Availability and scheduling | `DATABASE_SCHEMA.md`, `BACKEND_ARCHITECTURE.md`, `PLAN.md` | `apps/web/src/components/booking/TimeSlotPicker.tsx`, `packages/shared/src/types/user.ts`, `packages/shared/src/constants/index.ts` | `availability|weekly|slot|expires|reminder|bonus` |
| Security hardening | `IMPLEMENTATION_GUARDRAILS.md`, `BACKEND_ARCHITECTURE.md`, `INFRASTRUCTURE_DEPLOYMENT.md`, `DATABASE_SCHEMA.md` | `backend/Dockerfile`, `backend/src/index.ts`, `apps/web/src/lib/api/client.ts` | `rate limit|WAF|RLS|Secrets|JWT|signature|circuit breaker` |
| Deployment and runtime ops | `INFRASTRUCTURE_DEPLOYMENT.md`, `backend/DOCKER.md` | `backend/Dockerfile`, `backend/docker-compose.yml`, `backend/package.json` | `ECS|ALB|RDS|SQS|HEALTHCHECK|migrate deploy` |
| Mobile customer flow | `PLAN.md`, `FRONTEND_ARCHITECTURE.md` | `apps/mobile/app/(customer)/*`, `apps/mobile/lib/api/client.ts` | `mock|placeholder|search|bookings|cleaner/[id]` |
| Mobile cleaner flow | `PLAN.md`, `FRONTEND_ARCHITECTURE.md` | `apps/mobile/app/(cleaner)/*` | `dashboard|earnings|tabs` |
| Shared validation/contracts | `API_SPECIFICATION.md`, `DATABASE_SCHEMA.md` | `packages/shared/src/schemas/validation.ts`, `packages/shared/src/types/*`, `packages/shared/src/constants/index.ts` | `zod|schema|BookingRequest|SearchCleaners|BOOKING_STATUS` |

---

## Practical Search Shortcuts

| Goal | Command |
|---|---|
| Find all TODOs in first-party code | `rg -n "TODO" apps backend packages` |
| Find where an endpoint is referenced | `rg -n "/bookings|/search/cleaners|/payments/create-intent" apps backend` |
| Find auth implementation points | `rg -n "Clerk|Bearer|auth_token|external_auth_id|role" apps backend packages *.md` |
| Find booking address model usage | `rg -n "address|location|street|zip|specialInstructions" apps backend packages *.md` |
| Find Stripe/webhook/idempotency logic | `rg -n "Stripe|webhook|idempot|payment_intent|payout|refund" apps backend *.md` |
| Find scheduler/worker/queue references | `rg -n "queue|worker|SQS|schedule|cron|expires" backend *.md` |
| Compare API docs vs implemented backend surface | `rg -n "GET|POST|PATCH|DELETE|/v1|/api/v1" API_SPECIFICATION.md backend/src` |

---

## Current Implementation Reality (Quick Read)

| Area | Current State |
|---|---|
| Backend API | Minimal Elysia app with health and status endpoints |
| Backend DB | Prisma configured with `User`, `Cleaner`, `Booking` models |
| Web app | Route scaffolding and reusable components exist; many pages are placeholders/TODO |
| Mobile app | Expo route scaffolding exists; major screens currently use mock data/placeholders |
| Shared package | Types/schemas/constants are present, useful as contract baseline |

Use this file as the first stop, then jump to the mapped docs/files for deep work.
