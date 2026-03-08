# Sweep MVP Plan v2

## Overview

This is the working implementation plan for Sweep MVP. It is an actual-status tracker, not a simple aspirational roadmap.

- MVP scope is **web-first**.
- `apps/mobile/` stays post-MVP unless explicitly promoted later.
- A slice is only `Complete` when live backend behavior, live web behavior, required tests, and launch guardrails are all in place.
- Work already built out of order stays in the plan, but is labeled honestly as `Partial` or `Scaffold` until it meets completion criteria.

**Reference docs:** [SYSTEM_DESIGN_OVERVIEW.md](./SYSTEM_DESIGN_OVERVIEW.md) | [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) | [API_SPECIFICATION.md](./API_SPECIFICATION.md) | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) | [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md) | [TESTING_STRATEGY.md](./TESTING_STRATEGY.md)

**Detailed execution notes:** [docs/plans/2026-02-10-implementation-execution-plan.md](./docs/plans/2026-02-10-implementation-execution-plan.md)

---

## Status Legend

- **Complete**: live backend behavior, live web flow, required tests, and rollout/guardrail criteria are satisfied.
- **Partial**: some real implementation exists, but at least one launch-critical requirement is still missing.
- **Scaffold**: route/page/component exists or mock behavior exists, but the core business behavior is not real yet.
- **Not started**: no meaningful implementation exists yet.

---

## Current Baseline

| Area | Status | Current reality |
|---|---|---|
| Track 0 - Contract + Launch Foundation | Not started | Core mismatches still exist across shared contracts, backend behavior, web behavior, and repo guardrails. |
| Slice 1 - Cleaner Onboarding | Partial | Backend onboarding, availability, Stripe Connect bootstrap, and web cleaner profile/calendar flows are real. Hardening and live end-to-end evidence are still missing. |
| Slice 2 - Customer Search & Browse | Partial | Search, cleaner profile, review read path, and public availability are real. Date-aware search and live availability-driven slot selection are still missing. |
| Slice 3 - Booking Flow | Scaffold | Customer booking UI exists and backend create/cancel routes exist, but there are no holds, no conflict prevention, and no canonical confirm flow. |
| Slice 4 - Payments & Payouts | Scaffold | Stripe Connect exists, but payment intents, payment state transitions, job completion, payouts, and reconciliation are not real. |
| Slice 5 - Reviews & Ratings | Partial | Review read endpoints exist. Review creation, aggregate updates, and ranking integration do not. |
| Slice 6 - Bonus Pool & Transparency | Partial | Bonus leaderboard/summary and customer fee breakdown ideas exist. Authoritative bonus calculation, persistence, awards, and ops ownership do not. |
| Slice 7 - Admin & Operations | Partial | Admin dashboard/users/disputes surfaces exist. Dispute creation, refund execution, auditability, and runbooks are still missing. |

---

## Planning Rules

### Delivery Model

Every slice still follows the same five-phase model:

1. **Scaffold**
2. **Contract**
3. **Data + Logic**
4. **Integration UX**
5. **Hardening + Rollout**

The important change is status reporting: a slice can have code from later phases while still being only `Partial` overall.

### Launch Blocking Rule

`Track 0 - Contract + Launch Foundation` is a release blocker. No slice should be treated as complete for launch readiness until Track 0 is materially complete.

### Documentation Sync Rule

After each slice or track change, update the docs in the same PR:

- `PLAN.md`
- `API_SPECIFICATION.md`
- any affected architecture doc
- any affected testing or guardrail doc

### Canonical Build Rules

- Money values in API contracts are integer cents only.
- Clerk bearer tokens are the production auth model.
- `x-external-auth-id` is a local-development bridge, not a public production contract.
- Canonical customer booking flow is:
  `availability -> hold -> payment intent -> confirm`
- The current direct `POST /bookings` path is non-canonical until it is rewritten to honor real slot selection and conflict rules.
- Use `default branch` wording in planning docs until the repos are normalized around `main` versus `master`.

---

## Track 0: Contract + Launch Foundation

**Status:** Not started

**Goal:** Reconcile the contracts, implementations, and repo guardrails so new feature work is built on a stable base.

### Why this comes first

The current codebase has real progress, but it also has drift:

- shared contracts and live backend behavior disagree in places
- web flows mix real API calls with scaffolding and local-only success states
- `apps/web` is missing the GitHub guardrails already present in other repos
- booking and payment flows do not have a single canonical path yet

### Work required

1. Reconcile `packages/shared`, backend routes, and web API clients before adding new customer booking behavior.
2. Normalize money units to integer cents at the API boundary; format dollars only in UI code.
3. Make `hold -> payment intent -> confirm` the only canonical customer booking flow.
4. Mark direct booking creation as transitional until it wraps the canonical flow instead of bypassing it.
5. Update [API_SPECIFICATION.md](./API_SPECIFICATION.md) to show:
   - what is live today
   - what is partial or scaffolded
   - what the canonical MVP contract should be
6. Add missing `apps/web` repo guardrails:
   - CI
   - CodeQL
   - dependency review
   - CODEOWNERS
   - provenance workflow
   - branch/PR protections
7. Finish cross-cutting hardening already captured in [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md):
   - migration-only deploy workflow
   - default-deny auth and ownership enforcement
   - durable webhook idempotency using the database
   - async webhook processing and retries
   - alarms, bake windows, and rollback/runbooks
8. Add a documentation maintenance rule to the working process so the docs do not drift again.

### Done criteria

- Shared contracts, backend handlers, and web clients agree on money units and endpoint intent.
- Booking flow has a single documented canonical path.
- `apps/web` has the same minimum PR guardrails as the other repos.
- The API spec no longer claims the backend is only scaffolded.
- Webhook processing is documented as durable and async, not in-memory only.

---

## Slice 1: Cleaner Onboarding

**Status:** Partial

**Goal:** A cleaner can authenticate, create a real profile, configure availability, and start Stripe Connect onboarding from the web app.

### Real today

- User sync and `GET /auth/me` exist in backend.
- Cleaner profile create/update and cleaner profile read exist.
- Weekly availability and blackout persistence exist.
- Public cleaner availability query exists.
- Stripe Connect onboarding link creation exists.
- Stripe `account.updated` webhook handling exists for onboarding state.
- Web cleaner profile, calendar, dashboard, and earnings pages call live backend endpoints.

### Still missing before this slice is complete

- Profile photo upload/storage pipeline.
- Clear production-only bearer-token path for all cleaner flows.
- Durable webhook idempotency and async processing instead of in-memory dedupe.
- Live browser smoke coverage against a real backend; current Playwright coverage is mostly mocked.
- Rollout/monitoring evidence for Stripe onboarding completion and failure handling.

### Completion evidence

- Cleaner can complete onboarding from browser against live backend.
- Profile, services, schedule, and blocked dates persist correctly.
- Stripe onboarding status is durable across restarts and webhook replays.
- Automated tests cover validation, auth, ownership, and webhook replay behavior.

---

## Slice 2: Customer Search & Browse

**Status:** Partial

**Goal:** A customer can search for cleaners, open a cleaner profile, inspect reviews, and choose a real available slot.

### Real today

- `GET /search/cleaners` returns live search results.
- `GET /cleaners/:id` returns live cleaner profile data.
- `GET /cleaners/:id/reviews` returns live review read data.
- `GET /cleaners/:id/availability?date=&duration=` returns real slots.
- Web search page and customer cleaner profile page call the backend.

### Still missing before this slice is complete

- Search must honor `date` by excluding cleaners without valid availability on that date.
- The customer cleaner profile must use live availability instead of hardcoded time slots.
- Decide map behavior for MVP explicitly:
  - either real map integration
  - or list-first launch with the map kept behind a flag
- Add at least one live end-to-end customer flow:
  `search -> profile -> select slot`

### Completion evidence

- Search results change when `date` changes.
- Customer selects from real returned slots, not UI placeholders.
- List view is launch-ready; map is either real or clearly non-blocking.
- Live end-to-end customer browse flow is covered in test and manual verification.

---

## Slice 3: Booking Flow

**Status:** Scaffold

**Goal:** A customer can reserve a slot, carry that reservation through checkout, confirm the booking exactly once, and manage the resulting booking.

### What exists today

- Customer booking pages exist in the web app.
- Backend exposes direct booking create, booking read/list, and booking cancel routes.
- Pricing helpers and refund helpers exist in the backend domain layer.

### What is not real yet

- No booking hold creation route.
- No hold expiry handling.
- No idempotent booking confirm flow.
- No conflict prevention based on selected slot.
- Current direct `POST /bookings` does not honor the chosen slot and fabricates booking time.
- Customer checkout is still a frontend-only success simulation.
- Notification sending does not exist.

### Build requirements

1. Add `POST /booking/holds` with:
   - TTL
   - idempotency
   - slot conflict checks
   - pricing snapshot
2. Add `POST /booking/confirm` that confirms only from a valid hold plus successful payment state.
3. Keep `POST /bookings` only if it becomes a thin wrapper over the canonical flow.
4. Ensure booking list/detail/cancel behavior uses real lifecycle state.
5. Add notification triggers for booking creation and cancellation.
6. Rewire the web booking flow so it passes a hold identifier through checkout instead of raw querystring totals.

### Completion evidence

- Two customers cannot book the same slot.
- Expired holds cannot be confirmed.
- Duplicate idempotency keys behave correctly.
- Web customer flow is:
  `profile -> choose real slot -> create hold -> checkout -> confirm`

---

## Slice 4: Payments & Payouts

**Status:** Scaffold

**Goal:** Payment, completion, payout, and reconciliation work as a single reliable booking-money lifecycle.

### What exists today

- Stripe Connect onboarding bootstrap exists.
- Booking pricing helpers exist.
- Cleaner earnings/dashboard surfaces exist in the web app.

### What is not real yet

- `POST /payments/create-intent` is still stubbed.
- Payment records and payment state transitions are not fully implemented.
- `POST /bookings/:id/complete` does not exist.
- Customer confirm / auto-confirm does not exist.
- Payout creation and Stripe transfer execution do not exist.
- Reconciliation and retry flows do not exist.

### Build requirements

1. Make payment intent creation real Stripe-backed, or fold it into the canonical confirm flow.
2. Persist payment state and webhook events durably.
3. Add booking completion endpoint for cleaner action.
4. Add customer confirm or timed auto-confirm behavior.
5. Create payout records and Stripe transfers after successful completion.
6. Reconcile booking, payment, payout, and webhook states with replay-safe handling.
7. Update cleaner earnings/dashboard data to show actual payout lifecycle.

### Completion evidence

- Successful payment confirms a booking exactly once.
- Failed payment does not create or retain an invalid confirmed booking.
- Completed job eventually produces a payout record and Stripe transfer.
- Webhook replay does not duplicate payment or payout side effects.

---

## Slice 5: Reviews & Ratings

**Status:** Partial

**Goal:** Customers can leave reviews after completed bookings, and ratings affect discovery in a controlled way.

### Real today

- Cleaner profile review read path exists.
- Cleaner aggregate rating and review count fields exist in the database and API responses.

### Still missing before this slice is complete

- `POST /bookings/:id/review` creation path.
- One-review-per-booking enforcement in the booking lifecycle.
- Validation that only completed bookings can be reviewed.
- Aggregate rating updates when a new review is created.
- Search ranking integration that uses live review data.

### Completion evidence

- Completed bookings can be reviewed exactly once.
- New review updates aggregate rating and count.
- Search ranking uses rating without hiding new cleaners unfairly.
- Tests cover review permissions, duplicates, and aggregate update logic.

---

## Slice 6: Bonus Pool & Transparency

**Status:** Partial

**Goal:** The bonus pool is not just a marketing line item; it is a real, explainable, and auditable part of the platform economics.

### Real today

- Bonus leaderboard and cleaner bonus summary endpoints exist.
- Customer-facing fee breakdown concepts already appear in web booking flows.

### Still missing before this slice is complete

- Authoritative bonus period calculation from real completed-booking GMV.
- Persistent bonus awards tied to a period and cleaner.
- Distribution job/process for actual bonus awards.
- Admin or ops visibility into pool totals, period state, and award history.
- Clear operational ownership for recalculation, adjustments, and disputes.

### Completion evidence

- Bonus pool amount is derived from real GMV, not placeholder data.
- Award history is persisted and queryable.
- Cleaner-facing bonus summaries reflect real stored periods and awards.
- Ops can explain how a period was calculated and distributed.

---

## Slice 7: Admin & Operations

**Status:** Partial

**Goal:** Admin tools can support a live marketplace, including disputes, user management, financial triage, and operational visibility.

### Real today

- Admin dashboard route exists.
- Admin user listing and user suspension endpoint exist.
- Admin dispute listing and resolve endpoint exist.

### Still missing before this slice is complete

- Customer-facing `POST /bookings/:id/dispute` creation flow.
- Actual refund execution tied to dispute resolution.
- Date-filtered/admin-grade stats instead of a single fixed summary.
- Suspension effects enforced in customer search and booking eligibility.
- Audit log and booking state transition visibility for support actions.
- Runbooks for dispute handling, refund failures, and manual replay/recovery.

### Completion evidence

- Admin can see, create, resolve, and audit disputes end to end.
- Refund and re-clean decisions trigger the correct downstream actions.
- Suspended users are blocked from the relevant product surfaces.
- Ops has enough state visibility to debug live incidents without raw database access.

---

## Test Plan

### Baseline to preserve

Keep the currently passing automated suites in:

- `backend`
- `packages/shared`
- `apps/web`
- `apps/mobile`

### New tests required

1. Shared contract tests for:
   - hold creation
   - booking confirm
   - booking complete
   - booking review
   - booking dispute
   - refund behavior
   - money-unit consistency
2. Backend integration tests for:
   - expired holds
   - duplicate idempotency keys
   - double-booking prevention
   - webhook replay
   - payout creation
   - refund execution
   - rating aggregation
3. Live web end-to-end coverage for launch blockers:
   - cleaner onboarding
   - customer search/profile with real availability
   - hold -> payment intent -> confirm
   - cancellation/refund
   - cleaner complete -> customer confirm -> payout

### Rule for mocked e2e tests

Mocked Playwright tests are still useful for fast UI checks, but mocked tests alone do not count as slice-complete evidence.

---

## Assumptions And Defaults

- MVP launch remains web-first.
- `apps/mobile` is not a launch gate.
- Existing out-of-order work remains in place and is tracked honestly rather than hidden.
- Interactive map is not a launch blocker if list view plus distance sorting is solid.
- The backend may keep transitional routes temporarily, but new work should be built toward the canonical flow described here.
