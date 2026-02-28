# Sweep Engineering Playbook

This file defines the delivery architecture for agent-driven development.

## 1) Canonical Sources of Truth

When contracts conflict, resolve in this order:

1. `packages/shared/src/contracts/**` (API/domain contracts)
2. `backend/prisma/schema.prisma` (persistence contract)
3. `API_SPECIFICATION.md` + `DATABASE_SCHEMA.md` (human-readable spec)

Do not introduce endpoint or status variants outside these files.

## 2) Backend Architecture Layers

- `backend/src/domain/**`: pure business logic (pricing, transitions, idempotency)
- `backend/src/modules/**` (next step): use-cases and orchestration
- `backend/src/adapters/**` (next step): HTTP/queue/storage integrations

Rules:
- Domain layer must not import framework/runtime adapters.
- Route handlers should be thin (validation + auth + orchestration only).
- DB writes for mutating endpoints must be idempotent.

## 3) Schema Design Principles

- Money values are integer cents only.
- Booking state machine is explicit (`confirmed`, `in_progress`, `completed`, `cancelled_by_customer`, `cancelled_by_cleaner`, `disputed`).
- Every mutating API call uses an idempotency key.
- Every state change is auditable (`booking_state_transitions`, `audit_logs`).

## 4) Agent Task Contract

Every implementation task must include:
- Goal
- Exact files to edit
- Acceptance criteria
- Required tests
- Rollback plan (if production-impacting)

## 5) Definition of Done (Per Change)

A change is done only when:
- Types compile
- Unit tests pass
- New behavior has tests for happy path + edge cases + failure path
- Contract/schema updates are reflected in shared contracts and Prisma
