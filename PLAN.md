# Sweep - Implementation Plan

## Overview

Concise implementation roadmap organized by MVP slices. Each slice delivers a complete end-to-end feature.

**Reference docs:** [SYSTEM_DESIGN_OVERVIEW.md](./SYSTEM_DESIGN_OVERVIEW.md) | [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) | [API_SPECIFICATION.md](./API_SPECIFICATION.md) | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) | [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md)

**Expanded execution plan (Feb 10, 2026):** [docs/plans/2026-02-10-implementation-execution-plan.md](./docs/plans/2026-02-10-implementation-execution-plan.md)

---

## Progress

- [ ] Cross-cutting Guardrails Track (security, deploy, reliability)
- [ ] Slice 1: Cleaner Onboarding
- [ ] Slice 2: Customer Search & Browse
- [ ] Slice 3: Booking Flow
- [ ] Slice 4: Payments & Payouts
- [ ] Slice 5: Reviews & Ratings
- [ ] Slice 6: Admin & Operations

---

## Delivery Model (Scaffolded, Not One-Shot)

This plan follows an incremental rollout model: every feature ships in phases with test gates between phases.

### Five-Phase Build Pattern (Use for Every Feature)

1. **Phase A - Scaffold**
   - Add route/page/handler skeleton and placeholder UI.
   - Keep feature behind a flag or hidden route if incomplete.
   - Goal: mergeable structure with no production behavior change.

2. **Phase B - Contract**
   - Lock request/response schema and validation.
   - Add contract tests for happy path + key error cases.
   - Goal: API shape and data contracts are stable before heavy logic.

3. **Phase C - Data + Logic**
   - Add DB changes (Prisma migration), business logic, and permissions.
   - Add integration tests against real DB/services (or test doubles).
   - Goal: feature works end-to-end in local and staging.

4. **Phase D - Integration UX**
   - Wire real frontend/mobile flows (remove mock placeholders).
   - Add e2e tests for core journey.
   - Goal: user-visible behavior is validated in staging.

5. **Phase E - Hardening + Rollout**
   - Add monitoring, alerts, retries, idempotency, and failure-path tests.
   - Roll out with canary percentages and rollback criteria.
   - Goal: safe production rollout with observability and quick recovery.

### Promotion Gates (Do Not Skip)

| Gate | Required Evidence |
|---|---|
| A -> B | Scaffold merged, feature flag defined, no regressions in lint/typecheck |
| B -> C | Contract tests green, API schema reviewed |
| C -> D | Integration tests green, auth/permission checks verified |
| D -> E | e2e path green in staging, manual QA checklist completed |
| E -> GA | Canary metrics healthy, alerts clean, rollback path validated |

### Testing Ladder (Minimum)

- **Unit tests** for pure business logic.
- **Contract tests** for endpoint payloads and validation errors.
- **Integration tests** for DB + service interactions.
- **E2E tests** for primary user journey per slice.
- **Manual checkpoint** after each slice before promoting to next slice.

### Rollout Strategy

- Ship incomplete work behind feature flags.
- Roll out by percentage or tenant cohort: `10% -> 50% -> 100%`.
- Define rollback triggers before release:
  - error rate spike
  - checkout/payment failure increase
  - queue/DLQ growth
  - latency SLO breach

---

## Cross-cutting Guardrails Track

**Goal:** Establish production guardrails early so feature slices ship safely.

- [ ] Migration-only production workflow (`prisma migrate deploy` in pipeline)
- [ ] Strict auth middleware and authorization policy on all protected routes
- [ ] Stripe webhook hardening (signature verify + idempotent store + async worker)
- [ ] ECS deployment guardrails (circuit breaker rollback + alarms)
- [ ] Reliability drill cadence (failover, queue/DLQ, webhook replay)

**Execution details:** See [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md)

---

## Slice 1: Cleaner Onboarding

**Goal:** A cleaner can sign up, set their profile, configure pricing/availability, and connect Stripe.

### Slice 1 Rollout Phases

- [ ] Phase A: Scaffold onboarding routes/forms and placeholder API wiring
- [ ] Phase B: Lock auth/profile/availability contracts and validation tests
- [ ] Phase C: Implement DB + backend logic + integration tests
- [ ] Phase D: Connect full web flow and run e2e onboarding journey
- [ ] Phase E: Harden Stripe onboarding + canary rollout with monitoring

### 1.1 Database & Backend Setup

- [ ] Create cleaner tables (users, cleaners, availability)
  - Done when: migrations run, schema matches DATABASE_SCHEMA.md
  - Tests:
    - `test: cleaners table has required columns`
    - `test: availability table supports recurring schedules`

- [ ] Auth integration (Clerk + API sync)
  - Done when: cleaner can authenticate via Clerk and API role checks pass
  - Tests:
    - `test: missing bearer token returns 401 on protected endpoints`
    - `test: POST /auth/sync upserts user with cleaner role`
    - `test: role mismatch returns 403 on cleaner endpoints`

### 1.2 Profile Setup

  - [ ] Profile API (create, update, get)
  - Done when: cleaner can set bio, photo, services, service area
  - Tests:
    - `test: POST /cleaner/profile updates bio`
    - `test: profile photo uploads to S3, stores signed URL`
    - `test: service area validates SF zip codes`

- [ ] Pricing configuration
  - Done when: cleaner can set hourly rate ($25-150 range)
  - Tests:
    - `test: rate below $25 returns validation error`
    - `test: rate persists and returns in profile GET`

### 1.3 Availability Setup

- [ ] Availability API (set weekly schedule, block dates)
  - Done when: cleaner can set recurring weekly hours and block specific dates
  - Tests:
    - `test: POST /cleaner/availability sets Mon-Fri 9am-5pm`
    - `test: overlapping time slots are rejected`
    - `test: blocked date removes availability for that day`

- [ ] Availability query endpoint
  - Done when: can query cleaner's open slots for a date range
  - Tests:
    - `test: GET /cleaner/:id/availability?start=&end= returns open slots`
    - `test: blocked dates not returned as available`

### 1.4 Stripe Connect

- [ ] Stripe Connect onboarding flow
  - Done when: cleaner can connect Stripe account for payouts
  - Tests:
    - `test: POST /cleaner/stripe/connect returns Stripe onboarding URL`
    - `test: webhook updates cleaner stripe_connected status`
    - `test: cleaner without Stripe cannot be booked`

### 1.5 Frontend (Web)

- [ ] Cleaner signup flow (pages: register, verify email, profile setup wizard)
  - Done when: cleaner can complete onboarding end-to-end in browser
  - Tests:
    - `test: e2e - cleaner completes signup wizard`
    - `test: form validation shows inline errors`
    - `test: progress persists if user leaves mid-wizard`

### Slice 1 Checkpoint

**Manually verify:** Create a test cleaner account, complete profile, set $50/hr rate, set availability for next week, connect Stripe test account. Confirm profile appears complete in database.

---

## Slice 2: Customer Search & Browse

**Goal:** A customer can sign up, search for cleaners by location/date, and view cleaner profiles.

### Slice 2 Rollout Phases

- [ ] Phase A: Scaffold search/list/profile pages with mock data
- [ ] Phase B: Lock search/profile API contracts and filter semantics
- [ ] Phase C: Implement geo search + availability filtering in backend
- [ ] Phase D: Wire real web search UX (filters, map/list, profile)
- [ ] Phase E: Harden ranking behavior + canary rollout and metrics checks

### 2.1 Customer Auth

- [ ] Customer auth (Clerk session + platform role)
  - Done when: customer can authenticate and access customer-only endpoints
  - Tests:
    - `test: POST /auth/sync with role=customer upserts customer`
    - `test: customer cannot access /cleaner/* endpoints`

### 2.2 Search API

- [ ] Search cleaners endpoint (location, date, filters)
  - Done when: returns available cleaners sorted by relevance
  - Tests:
    - `test: GET /search/cleaners?lat=37.7749&lng=-122.4194&date=2024-02-01 returns available cleaners`
    - `test: cleaner with no availability on date not returned`
    - `test: cleaner outside service area not returned`
    - `test: results include rating, rate, distance`

- [ ] Filter & sort options
  - Done when: can filter by price range, rating, services offered
  - Tests:
    - `test: filter min_rate=40&max_rate=60 excludes $35/hr cleaner`
    - `test: sort=rating_desc returns highest rated first`

### 2.3 Cleaner Public Profile

- [ ] Public profile endpoint
  - Done when: customers can view cleaner bio, photos, reviews, availability
  - Tests:
    - `test: GET /cleaners/:id returns public profile (no private data)`
    - `test: includes aggregate rating and review count`

### 2.4 Frontend (Web)

- [ ] Search page with map/list view
  - Done when: customer can search, filter, and browse results
  - Tests:
    - `test: e2e - search by zip returns results`
    - `test: clicking cleaner card opens profile page`

- [ ] Cleaner profile page
  - Done when: shows full profile with availability calendar
  - Tests:
    - `test: e2e - profile shows bio, rate, available dates`

### Slice 2 Checkpoint

**Manually verify:** As a customer, search for cleaners in 94110, filter to $40-60/hr, view a cleaner profile, see their available dates next week.

---

## Slice 3: Booking Flow

**Goal:** A customer can select a cleaner, choose date/time/services, and create a booking.

### Slice 3 Rollout Phases

- [ ] Phase A: Scaffold booking draft flow and slot-selection UI
- [ ] Phase B: Lock booking/hold API contracts and idempotency behavior
- [ ] Phase C: Implement holds, conflict prevention, and cancellation logic
- [ ] Phase D: Connect checkout journey end-to-end in staging
- [ ] Phase E: Harden concurrency/failure handling + canary rollout

### 3.1 Booking API

- [ ] Create booking endpoint
  - Done when: customer can create a hold on a cleaner's time slot
  - Tests:
    - `test: POST /bookings creates booking with status=confirmed`
    - `test: booking on unavailable slot returns 409`
    - `test: double-booking same slot returns 409`
    - `test: booking calculates total (rate x hours + 8% fee)`

- [ ] Booking management endpoints
  - Done when: can view, cancel bookings
  - Tests:
    - `test: GET /me/bookings returns customer's bookings`
    - `test: GET /cleaner/dashboard returns cleaner upcoming bookings`
    - `test: POST /bookings/:id/cancel cancels if >24hrs before`

### 3.2 Availability Locking

- [ ] Temporary hold system (5 min reservation during checkout)
  - Done when: slot is held while customer completes payment
  - Tests:
    - `test: held slot not bookable by another customer`
    - `test: hold expires after 5 min, slot released`

### 3.3 Notifications

- [ ] Booking notification triggers (email)
  - Done when: cleaner notified of new booking, customer gets confirmation
  - Tests:
    - `test: booking created sends email to cleaner`
    - `test: booking confirmed sends email to customer`

### 3.4 Frontend (Web)

- [ ] Booking flow (select date/time -> services -> review -> checkout)
  - Done when: customer can complete booking from cleaner profile
  - Tests:
    - `test: e2e - select slot, choose services, proceed to payment`
    - `test: unavailable slots are disabled in calendar`

### Slice 3 Checkpoint

**Manually verify:** As customer, book a cleaner for 3 hours next Tuesday at $55/hr. Confirm booking shows $178.20 total (with 8% fee). Check cleaner received email notification.

---

## Slice 4: Payments & Payouts

**Goal:** Customer pays at booking, cleaner receives payout after job completion.

### Slice 4 Rollout Phases

- [ ] Phase A: Scaffold payment and earnings UI with placeholders
- [ ] Phase B: Lock payment/payout/webhook contracts and event states
- [ ] Phase C: Implement Stripe intent/webhook/payout logic with idempotency
- [ ] Phase D: Connect real payment UX and cleaner earnings screens
- [ ] Phase E: Harden reconciliation/retry paths + controlled rollout

### 4.1 Payment Processing

- [ ] Stripe payment intent on booking
  - Done when: customer card charged when booking confirmed
  - Tests:
    - `test: booking confirmation creates Stripe charge`
    - `test: failed payment does not create confirmed booking and notifies customer`
    - `test: successful payment updates booking to confirmed`

- [ ] Payment webhook handling
  - Done when: Stripe webhooks update payment status reliably
  - Tests:
    - `test: payment_intent.succeeded webhook marks booking paid`
    - `test: webhook idempotency - duplicate events handled`

### 4.2 Job Completion

- [ ] Job completion flow
  - Done when: cleaner marks done, customer confirms (or auto-confirm 24hr)
  - Tests:
    - `test: POST /bookings/:id/complete sets status=completed`
    - `test: customer confirm releases payout`
    - `test: auto-confirm triggers after 24hr if no disputes`

### 4.3 Payouts

- [ ] Stripe Connect payout to cleaner
  - Done when: cleaner receives transfer after job confirmed
  - Tests:
    - `test: payout creates Stripe transfer to cleaner account`
    - `test: payout amount = cleaner rate x hours (no platform fee deducted)`
    - `test: payout records stored in payouts table`

### 4.4 Frontend (Web)

- [ ] Payment form (Stripe Elements)
  - Done when: customer can enter card and pay securely
  - Tests:
    - `test: e2e - enter test card, payment succeeds`
    - `test: invalid card shows error`

- [ ] Cleaner earnings dashboard
  - Done when: cleaner can view pending/completed payouts
  - Tests:
    - `test: e2e - cleaner sees payout history`

### Slice 4 Checkpoint

**Manually verify:** Complete a booking with Stripe test card. Mark job complete as cleaner. Confirm as customer. Verify Stripe dashboard shows transfer to cleaner's connected account.

---

## Slice 5: Reviews & Ratings

**Goal:** Customers can review cleaners after jobs, ratings affect search ranking.

### Slice 5 Rollout Phases

- [ ] Phase A: Scaffold review submission/list UI
- [ ] Phase B: Lock review/rating contracts and constraints
- [ ] Phase C: Implement write path + aggregate rating updates
- [ ] Phase D: Wire profile/search UX with live ratings
- [ ] Phase E: Harden abuse/reporting + rollout with ranking regression checks

### 5.1 Reviews API

- [ ] Create review endpoint
  - Done when: customer can submit rating (1-5) and text review after completed job
  - Tests:
    - `test: POST /bookings/:id/review creates review`
    - `test: review only allowed after job completed`
    - `test: one review per booking enforced`
    - `test: rating must be 1-5`

- [ ] Review retrieval
  - Done when: reviews appear on cleaner profile
  - Tests:
    - `test: GET /cleaners/:id/reviews returns paginated reviews`
    - `test: reviews sorted by newest first`

### 5.2 Rating Aggregation

- [ ] Calculate and cache cleaner ratings
  - Done when: average rating updates after each review
  - Tests:
    - `test: new review updates cleaner average_rating`
    - `test: rating calculated from last 50 reviews (recency weighted)`

### 5.3 Search Ranking Integration

- [ ] Factor ratings into search results
  - Done when: higher-rated cleaners rank higher (with availability)
  - Tests:
    - `test: search default sort factors in rating`
    - `test: new cleaner without reviews still appears`

### 5.4 Frontend (Web)

- [ ] Review submission form (post-booking)
  - Done when: customer prompted to review after job completion
  - Tests:
    - `test: e2e - submit 5-star review with comment`

- [ ] Reviews display on profile
  - Done when: cleaner profile shows reviews with pagination
  - Tests:
    - `test: e2e - profile shows rating average and review list`

### Slice 5 Checkpoint

**Manually verify:** Complete a booking flow. Submit a 5-star review as customer. Check cleaner profile shows the review and updated rating. Search results reflect rating.

---

## Slice 6: Admin & Operations

**Goal:** Admin can manage users, handle disputes, view metrics.

### Slice 6 Rollout Phases

- [ ] Phase A: Scaffold admin routes and dashboard shells
- [ ] Phase B: Lock admin/dispute/metrics API contracts
- [ ] Phase C: Implement RBAC, dispute resolution, and metrics queries
- [ ] Phase D: Connect admin UI workflows end-to-end
- [ ] Phase E: Harden auditability/runbooks + phased admin rollout

### 6.1 Admin Auth

- [ ] Admin role and protected routes
  - Done when: admin users can access /admin/* endpoints
  - Tests:
    - `test: non-admin gets 403 on /admin/*`
    - `test: admin can access all admin endpoints`

### 6.2 User Management

- [ ] View and manage users (cleaners, customers)
  - Done when: admin can list, search, suspend users
  - Tests:
    - `test: GET /admin/users returns paginated user list`
    - `test: POST /admin/users/:id/suspend deactivates user`
    - `test: suspended cleaner not returned in search`

### 6.3 Dispute Handling

- [ ] Dispute creation and resolution
  - Done when: customer can report issue, admin can resolve
  - Tests:
    - `test: POST /bookings/:id/dispute creates dispute`
    - `test: admin can issue refund, credit, or dismiss`
    - `test: refund triggers Stripe refund`

### 6.4 Metrics Dashboard

- [ ] Basic operational metrics
  - Done when: admin can view bookings, revenue, user counts
  - Tests:
    - `test: GET /admin/stats returns GMV, booking count, user counts`
    - `test: metrics filterable by date range`

### 6.5 Frontend (Web)

- [ ] Admin dashboard pages
  - Done when: admin can manage platform from web UI
  - Tests:
    - `test: e2e - admin logs in, views user list`
    - `test: e2e - admin resolves dispute with refund`

### Slice 6 Checkpoint

**Manually verify:** Log in as admin. View user list. Create a dispute on a completed booking. Resolve with partial refund. Verify Stripe shows refund.

---

## Post-MVP (Future Slices)

- [ ] Mobile app (React Native)
- [ ] SMS notifications
- [ ] Bonus system for top cleaners
- [ ] Background check integration
- [ ] Multi-city expansion
