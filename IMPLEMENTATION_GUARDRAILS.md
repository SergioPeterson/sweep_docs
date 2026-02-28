# Sweep - Implementation Guardrails

This document defines high-ROI, cross-cutting guardrails to implement before or during MVP feature work. It is implementation guidance only.

**Related docs:** [PLAN.md](./PLAN.md) | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | [INFRASTRUCTURE_DEPLOYMENT.md](./INFRASTRUCTURE_DEPLOYMENT.md)

---

## Recommended Execution Order

1. Migration-only production workflow
2. Strict auth middleware policy
3. Stripe webhook hardening (signature + idempotency + async worker)
4. ECS deploy guardrails (circuit breaker + alarms)
5. Reliability drills (failover, queue/DLQ, webhook replay)

---

## Feature Rollout Protocol (Applies to Every Slice)

- Build features in phases: `Scaffold -> Contract -> Data/Logic -> Integration UX -> Hardening/Rollout`.
- Keep unfinished work behind feature flags.
- Require objective promotion gates between phases (tests + manual checkpoint).
- Roll out in cohorts (`10% -> 50% -> 100%`) with predefined rollback triggers.

For task-level phase checklists, use `PLAN.md` as the execution guide.

---

## 1) Migration-Only Production Workflow (Prisma)

### Policy

- Production and staging schema changes must come from committed Prisma migrations (`backend/prisma/migrations/*`) only.
- `prisma migrate deploy` is the only migration command allowed in staging/production.
- `prisma db push` and `prisma migrate dev` are development-only commands and must never run against staging/production databases.

### CI/CD Requirements

- Pre-deploy checks:
  - `prisma validate`
  - `prisma generate`
  - `prisma migrate status`
- Deploy-time schema step:
  - Run `prisma migrate deploy` before app rollout.
- Deployment fails if migrations fail or drift is detected.

### Migration Safety Rules

- Use expand/contract migrations for breaking changes.
- Avoid destructive changes in a single deploy.
- Keep migration scripts deterministic and idempotent for reruns.
- Create DB snapshot/backup before production migration step.

### Done Criteria

- [ ] CI pipeline has an explicit migration stage that runs before service rollout.
- [ ] Staging/prod runbooks explicitly prohibit `db push` and `migrate dev`.
- [ ] Every schema PR includes generated migration files and rollback notes.
- [ ] Production deploy fails automatically on migration error.

---

## 2) Strict Auth Middleware Policy

### Policy

- Default-deny for API routes: every non-public route requires verified auth context.
- Route handlers must not parse/verify tokens directly; central middleware owns token verification.
- Authorization is mandatory on protected routes:
  - Role checks (customer/cleaner/admin)
  - Ownership/tenant boundary checks for resource access

### Middleware Contract

- Verify Clerk token using JWKS and validate issuer, audience, expiry, and not-before constraints.
- Map token subject (`sub`) to platform user (`users.external_auth_id`).
- Attach `authContext` to request:
  - `userId`
  - `externalAuthId`
  - `role`
  - `tenantId` (if multi-tenant dimension exists later)
- Return:
  - `401` for missing/invalid token
  - `403` for valid token without required role/ownership

### Route Classification

- Public routes:
  - health/liveness/readiness
  - API docs (if enabled)
  - Stripe webhook endpoint (signature-authenticated, not JWT-authenticated)
- Protected routes:
  - all customer/cleaner/admin API endpoints

### Done Criteria

- [ ] Single shared auth middleware/plugin verifies all bearer tokens.
- [ ] Every protected route has explicit role policy.
- [ ] Every resource fetch/update enforces ownership boundary (no cross-user access).
- [ ] Security tests exist for 401/403 and privilege escalation attempts.

---

## 3) Stripe Webhook Hardening

### Policy

- Webhook authenticity, idempotency, and asynchronous processing are mandatory.
- Webhook endpoint should acknowledge quickly after durable receipt.

### Required Design

- Signature verification:
  - Verify Stripe signature using raw request body and webhook secret.
  - Reject invalid signatures and timestamp out-of-tolerance events.
- Idempotent event store:
  - Persist Stripe `event.id` in a dedicated table with unique constraint.
  - Track state (`received`, `processing`, `processed`, `failed`) and attempt count.
- Async processing:
  - Ingress handler persists event and enqueues processing job.
  - Worker performs business logic and retry policy.
  - Poison events move to DLQ after retry limit.

### Consistency Rules

- Business mutations must be idempotent by natural keys (for example, payment intent ID or booking ID).
- Out-of-order event delivery must not corrupt state transitions.
- Duplicate events must be safe and side-effect free.

### Observability

- Emit metrics:
  - webhook verification failures
  - enqueue latency
  - processing success/failure count
  - retry count and DLQ count
- Alert on sustained webhook failures and DLQ growth.

### Done Criteria

- [ ] Raw-body signature verification implemented and tested.
- [ ] Unique event-id idempotency persisted before processing.
- [ ] Async worker path with retries and DLQ is in place.
- [ ] Replay tests prove duplicate and reordered events are safe.

---

## 4) ECS Deploy Guardrails (Circuit Breaker + Alarms)

### Policy

- No blind deployments: every rollout is health-gated and alarm-gated.
- Failed deployments must auto-rollback without manual SSH/container intervention.

### Required ECS/ALB Configuration

- ECS service deployment circuit breaker:
  - `enable = true`
  - `rollback = true`
- Rolling deployment healthy thresholds configured to avoid total outage during rollout.
- ALB target group health check uses readiness endpoint (`/readyz`) instead of generic liveness.

### Required Alarm Set

- ALB target 5XX error alarm.
- ALB target response time alarm (p95/p99).
- ECS service unhealthy/running task count below desired.
- ECS CPU and memory saturation alarms.
- Queue/DLQ alarms for worker pipelines.

### Deployment Gating

- Deployment pipeline must block promotion if critical alarms are firing.
- Post-deploy bake window (for example 10-15 minutes) before considering rollout successful.

### Done Criteria

- [ ] ECS circuit breaker rollback is enabled for API and worker services.
- [ ] Target group health checks point to readiness endpoint.
- [ ] CloudWatch alarms are wired and actionable.
- [ ] Runbook documents automatic rollback behavior and manual fallback steps.

---

## 5) Reliability Drills Program

### Cadence

- Run monthly in staging and quarterly in production-like conditions.
- Record outcome, MTTR, and follow-up actions in an ops log.

### Drill A: Failover Drill

- Objective:
  - Validate service continuity during database failover.
- Scenario:
  - Trigger controlled RDS failover / primary disruption.
- Pass criteria:
  - API recovers within target RTO.
  - No permanent data loss (RPO target maintained).
  - Alerting and rollback paths trigger as expected.

### Drill B: Queue Retry + DLQ Drill

- Objective:
  - Validate retry policy and poison message handling.
- Scenario:
  - Inject intentionally failing jobs and observe retry attempts.
- Pass criteria:
  - Retry backoff works as configured.
  - Jobs exceeding retry limit land in DLQ.
  - Recovery runbook can replay/fix DLQ safely.

### Drill C: Webhook Replay Drill

- Objective:
  - Validate webhook idempotency and duplicate/out-of-order safety.
- Scenario:
  - Replay Stripe events (duplicate and reordered) against staging.
- Pass criteria:
  - No duplicate side effects (charges, payouts, booking transitions).
  - Event processor remains consistent and auditable.

### Done Criteria

- [ ] Written runbook exists for each drill.
- [ ] Drill schedule exists with owners.
- [ ] Evidence (logs, metrics, incident notes) stored per run.
- [ ] Follow-up issues tracked to closure.

---

## Definition of Complete for This Guardrails Track

- [ ] All five sections above have implementation owners and target dates.
- [ ] Each section has tests/drills that run on a defined cadence.
- [ ] Guardrails are treated as release blockers for production launch readiness.
