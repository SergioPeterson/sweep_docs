# Sweep MVP Plan v2

## Overview

This is the working implementation plan for Sweep MVP. It is an actual-status tracker, not a simple aspirational roadmap.

- MVP scope is **web-first**.
- `apps/mobile/` stays post-MVP unless explicitly promoted later.
- A slice is only `Complete` when live backend behavior, live web behavior, required tests, and launch guardrails are all in place.
- Work already built out of order stays in the plan, but is labeled honestly as `Partial` or `Scaffold` until it meets completion criteria.

**Reference docs:** [SYSTEM_DESIGN_OVERVIEW.md](./SYSTEM_DESIGN_OVERVIEW.md) | [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) | [API_SPECIFICATION.md](./API_SPECIFICATION.md) | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md) | [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md) | [TESTING_STRATEGY.md](./TESTING_STRATEGY.md) | [AGENT_API_ARCHITECTURE.md](./AGENT_API_ARCHITECTURE.md)

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
| Slice 8 - Agent API (MCP Server) | Not started | MCP/agent access planning exists in docs, but OAuth, capability grants, approval flows, and agent-safe payment handling are not implemented yet. |

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

## Slice 8: Agent API (MCP Server)

**Status:** Not started

**Goal:** AI agents (Claude, ChatGPT, etc.) can connect to a user's Sweep account via MCP, search for cleaners, create bookings, and make payments on the user's behalf — securely, with user-controlled spending limits and approval gates.

**Design doc:** [AGENT_API_ARCHITECTURE.md](./AGENT_API_ARCHITECTURE.md)

### Slice 8 Rollout Phases

- [ ] Phase A: Scaffold MCP endpoint path, protected-resource metadata, registered-client auth, and read-only tool stubs
- [ ] Phase B: Lock Clerk OAuth verification, Sweep capability grants, transport/security contracts, and auth tests
- [ ] Phase C: Implement read-only tools, agent grants/sessions, and audit logging
- [ ] Phase D: Implement booking mutation tools with approval snapshots and revalidation
- [ ] Phase E: Implement payment tools, spending limits, gateway hardening, and optional preview-gated SPT adapter

### 8.1 MCP Server Scaffold & OAuth

- [ ] Streamable HTTP transport endpoint (`/mcp`)
  - Done when: MCP clients can connect, perform initialization handshake, and discover tools
  - Tests:
    - `test: POST /mcp returns valid JSON-RPC initialize response`
    - `test: unauthenticated request returns 401 with WWW-Authenticate header`
    - `test: session ID assigned on initialization and required on subsequent requests`
    - `test: invalid Origin header on GET/POST /mcp returns 403`

- [ ] OAuth 2.1 via Clerk
  - Done when: agent completes Authorization Code + PKCE flow, receives a Clerk OAuth access token, and can call tools after token verification plus resource/audience checks
  - Tests:
    - `test: GET /.well-known/oauth-protected-resource/mcp returns valid metadata pointing to Clerk`
    - `test: valid Clerk OAuth token grants access to MCP tools`
    - `test: valid Clerk token with wrong audience/resource binding returns 401`
    - `test: expired token returns 401`
    - `test: revoked token returns 401`

- [ ] Registered agent clients + first-party capability grants
  - Done when: approved MCP clients can connect through Clerk and Sweep enforces app-level capabilities independent of OAuth scopes
  - Tests:
    - `test: unsupported client_id is rejected before tool execution`
    - `test: consent screen displayed to user on first authorization`
    - `test: valid token without payments.charge capability returns 403 AGENT_CAPABILITY_REQUIRED`
    - `test: revoking local agent grant blocks a still-valid token immediately`

### 8.2 Read-Only Tools

- [ ] `search_cleaners` — search by location, date, service type, price range, rating
  - Done when: agent can search and receive structured results
  - Tests:
    - `test: search_cleaners with lat/lng returns available cleaners`
    - `test: search_cleaners requires marketplace.read capability`
    - `test: user-generated content in results is sanitized`

- [ ] `get_cleaner_profile` — full profile with reviews, rates, badges
  - Done when: agent receives cleaner details matching public profile API
  - Tests:
    - `test: get_cleaner_profile returns structured profile (no private data)`

- [ ] `check_availability` — check specific cleaner's open slots for a date range
  - Done when: agent can check slots before attempting to book
  - Tests:
    - `test: check_availability returns open slots for valid date`
    - `test: check_availability for past date returns error`

- [ ] `get_booking_quote` — calculate price for cleaner + service + duration
  - Done when: agent receives breakdown (subtotal, platform fee, total)
  - Tests:
    - `test: get_booking_quote calculates correct total with 8% fee`

- [ ] `get_my_bookings` — list user's upcoming, past, and cancelled bookings
  - Done when: agent can see the authenticated user's booking history
  - Tests:
    - `test: get_my_bookings returns only the authenticated user's bookings`
    - `test: get_my_bookings supports confirmed, in_progress, completed, cancelled, and disputed status filtering`

### 8.3 Mutation Tools (Booking)

- [ ] `create_booking` — create and hold a booking (returns details for user review)
  - Done when: agent can create a standard 5-minute booking hold for auto-approved requests and a pending approval snapshot for Tier 3 requests
  - Tests:
    - `test: create_booking requires bookings.create capability`
    - `test: create_booking within spending cap creates a 5-minute hold (Tier 2)`
    - `test: create_booking above spending cap returns pending_approval without creating a hold (Tier 3)`
    - `test: approving a pending request revalidates availability and creates a hold atomically`
    - `test: approval fails with booking_conflict if slot was taken before approval`
    - `test: approval fails with needs_requote if recalculated total exceeds approved amount`
    - `test: first-time cleaner triggers Tier 3 approval even within spending cap`
    - `test: create_booking on unavailable slot returns 409`
    - `test: idempotency key prevents duplicate bookings`

- [ ] `cancel_booking` — cancel an existing booking
  - Done when: agent can cancel with soft confirmation
  - Tests:
    - `test: cancel_booking requires bookings.manage capability`
    - `test: cancel_booking >24hrs before returns full refund amount`
    - `test: cancel_booking on non-owned booking returns 403`

- [ ] `reschedule_booking` — modify date/time of existing booking
  - Done when: agent can reschedule with availability and conflict checks
  - Tests:
    - `test: reschedule_booking requires bookings.manage capability`
    - `test: reschedule_booking checks new slot availability`
    - `test: reschedule_booking recalculates pricing if duration changes`
    - `test: idempotency key prevents duplicate reschedules`

- [ ] Tiered approval system
  - Done when: Tier 0-1 auto-approve, Tier 2 within-cap auto-approves, Tier 3 creates an approval snapshot and push notification, Tier 4 remains future-facing and requires interactive approval plus reverification
  - Tests:
    - `test: booking within per-transaction cap returns approval_status=approved`
    - `test: booking above cap creates pending approval and sends push notification`
    - `test: pending approval expires after 15 minutes`
    - `test: user deny via push notification returns structured denial to agent`
    - `test: approval notification explains that price and availability are revalidated on approval`

### 8.4 Payment Tools

- [ ] `confirm_and_pay_booking` — confirm held booking and process payment using Sweep's existing saved-payment-method flow
  - Done when: agent can complete payment using the customer's saved Stripe payment method, subject to approval checks and safe handoff when SCA or manual payment is required
  - Tests:
    - `test: confirm_and_pay_booking requires payments.charge capability`
    - `test: payment amount is recalculated server-side (never trust agent-provided amount)`
    - `test: successful payment transitions booking to confirmed`
    - `test: failed payment does not confirm booking`
    - `test: payment_intent.requires_action returns requires_customer_payment without confirming booking`
    - `test: expired hold cannot be paid`

- [ ] Spending limits
  - Done when: per-transaction, daily, and weekly aggregate limits are enforced server-side during both hold creation and payment confirmation
  - Tests:
    - `test: per-transaction cap blocks single booking above limit`
    - `test: daily aggregate cap blocks when cumulative agent spend exceeds limit`
    - `test: weekly aggregate cap blocks when rolling 7-day spend exceeds limit`
    - `test: payment confirmation re-checks projected aggregate spend before charging`
    - `test: user can configure custom spending limits via web/mobile`

- [ ] `rate_cleaner` — submit rating and review after completed booking
  - Done when: agent can submit a review on behalf of user
  - Tests:
    - `test: rate_cleaner requires reviews.write capability`
    - `test: rate_cleaner only allowed after job completed`
    - `test: one review per booking enforced`

- [ ] Optional preview-gated Stripe Shared Payment Token adapter
  - Done when: approved preview partners can pass a granted SPT to Sweep behind a feature flag without changing the default payment path
  - Tests:
    - `test: SPT path is disabled unless feature flag is on`
    - `test: valid granted SPT can be used to create a payment intent`
    - `test: invalid or expired granted SPT returns structured payment error`

### 8.5 Security & Observability

- [ ] Input validation and sanitization on all tool parameters
  - Done when: Zod schemas validate every tool input; user-generated content sanitized before returning to agents
  - Tests:
    - `test: malformed tool input returns structured validation error`
    - `test: cleaner bio with injection attempt is sanitized in search results`

- [ ] Rate limiting (per-user, per-session)
  - Done when: 60 reads/min, 10 writes/min per user session enforced
  - Tests:
    - `test: exceeding read rate limit returns 429 with retry-after`
    - `test: exceeding write rate limit returns 429`
    - `test: runaway loop detection (same tool, same params, >5 calls/min)`

- [ ] Transport and token hardening
  - Done when: MCP requests validate Origin, Clerk token verification includes resource/audience binding, and local grant status is checked on every request
  - Tests:
    - `test: valid token with wrong audience/resource metadata is rejected`
    - `test: revoked local agent grant returns 401 or 403 even if Clerk token has not expired`
    - `test: malformed MCP session id is rejected`

- [ ] Audit logging
  - Done when: every agent action produces an immutable audit record (who, what, when, session, outcome)
  - Tests:
    - `test: tool call creates audit log entry with user_id, session_id, tool_name, params, capability, result`
    - `test: payment tool calls include amount and booking_id in audit record`

- [ ] Instant revocation
  - Done when: user can revoke all agent access from web/mobile; revocation blocks tool execution immediately through local grant checks and clears active sessions
  - Tests:
    - `test: revoked authorization blocks the next tool call immediately`
    - `test: revoke-all clears every active agent session for the user`

### 8.6 Frontend (Web + Mobile)

- [ ] Agent access settings page
  - Done when: user can view connected agents, review granted capabilities, configure spending limits, and revoke access
  - Tests:
    - `test: e2e - user views connected agents list`
    - `test: e2e - user sees granted capabilities for an agent`
    - `test: e2e - user sets $150 per-transaction spending cap`
    - `test: e2e - user revokes agent access, agent is blocked on next call`

- [ ] Approval notification flow
  - Done when: Tier 3 actions trigger push notification with approve/deny, the server revalidates slot and price on approval, and the result propagates to the agent
  - Tests:
    - `test: e2e - agent books above cap, user receives push, approves, hold is created after revalidation`
    - `test: e2e - agent books above cap, user denies, agent receives structured denial`
    - `test: e2e - agent books above cap, slot changes before approval, user sees requote/conflict flow`

### Slice 8 Checkpoint

**Manually verify:** Connect an AI agent (approved test MCP client) to a user's Sweep account via OAuth. Search for cleaners, view a profile, get a quote. Book a cleaner within spending cap and verify a normal 5-minute hold is created. Book a cleaner above cap and verify the server creates an approval snapshot, sends a push notification, then revalidates availability and price before creating the hold after approval. Confirm payment with a saved Stripe payment method. Trigger a `requires_action` payment and verify handoff to normal checkout. Revoke agent access and verify the next tool call is blocked immediately.

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
