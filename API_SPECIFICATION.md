# API Specification

## Overview

This document now tracks two things at once:

1. the **live backend surface that exists today**
2. the **canonical MVP contract** the team should build toward next

The earlier version of this document described a mostly future-state API and incorrectly said the backend only exposed scaffold endpoints. That is no longer accurate.

### Status As Of March 8, 2026

- The backend has real onboarding, search/browse, bonus, cleaner dashboard, and partial admin surface area.
- Booking and payment flows are **not** yet canonical.
- Some live behavior still drifts from shared contracts and must be reconciled in `Track 0` from [PLAN.md](./PLAN.md).

### Base URL

- **Current live local/dev base URL:** `http://localhost:3050`
- **Canonical public API target:** `https://api.yourdomain.com/v1`

Current live backend routes are mostly unversioned. Normalizing around `/v1` is still pending.

### Status Legend

- **Live**: implemented and used by the app today.
- **Partial**: implemented, but missing important production or contract guarantees.
- **Scaffold**: route exists, but behavior is placeholder or non-canonical.
- **Planned**: required target endpoint that should exist, but does not yet.

---

## Canonical Rules

### Authentication

- Production contract is Clerk bearer-token auth.
- `Authorization: Bearer <access_token>` is the canonical auth mechanism.
- `x-external-auth-id` is a development bridge and should not be treated as a public production contract.

### Money Units

- Canonical API contract uses **integer cents** for all money fields.
- UI code is responsible for formatting dollars for display.
- Current live code still has drift here:
  - some web/backend search behavior uses dollar-style `maxRate`
  - this must be reconciled before new customer booking features are added

### Idempotency

Use `X-Idempotency-Key` on canonical mutating booking/payment/dispute endpoints.

### Canonical Booking Flow

The canonical customer path is:

`GET /cleaners/:id/availability -> POST /booking/holds -> POST /payments/create-intent -> POST /booking/confirm`

The current direct `POST /bookings` route is transitional and **non-canonical** until it is rewritten to honor real slot selection and conflict prevention.

---

## Current Live Surface

### Platform / Auth

| Endpoint | Status | Notes |
|---|---|---|
| `GET /health` | Live | Health check. |
| `GET /` | Live | Basic API info. |
| `GET /api/v1/status` | Live | Environment/config status. |
| `POST /auth/sync` | Live | Upserts platform user from Clerk identity. |
| `GET /auth/me` | Live | Returns current user when auth is present; returns null when absent/invalid. |

### Customer Search & Browse

| Endpoint | Status | Notes |
|---|---|---|
| `GET /search/cleaners` | Partial | Live search works, but `date` is not honored yet and cursor pagination is not implemented. |
| `GET /cleaners/:id` | Live | Returns public cleaner profile with embedded recent reviews. |
| `GET /cleaners/:id/reviews` | Live | Returns paginated review list and aggregate metadata. |
| `GET /cleaners/:id/availability?date=&duration=` | Live | Returns bookable slots for a date/duration; this should become the canonical slot source for the UI. |

### Cleaner

| Endpoint | Status | Notes |
|---|---|---|
| `GET /cleaner/profile` | Live | Returns cleaner profile for signed-in cleaner. |
| `POST /cleaner/profile` | Live | Creates or updates cleaner profile. |
| `POST /cleaner/availability` | Live | Stores weekly schedule. |
| `POST /cleaner/availability/blackouts` | Live | Stores blocked periods. |
| `POST /cleaner/stripe-connect` | Partial | Connect bootstrap is live; full durability/hardening is not done. |
| `GET /cleaner/dashboard` | Partial | Real data route exists, but downstream booking/payment lifecycle is incomplete. |
| `GET /cleaner/earnings` | Partial | Real summary route exists, but payout lifecycle is incomplete. |
| `GET /cleaner/bonuses/summary` | Partial | Summary exists, but bonus calculation is not authoritative yet. |

### Booking / Payment

| Endpoint | Status | Notes |
|---|---|---|
| `GET /me/bookings` | Partial | Read path exists, but lifecycle is based on non-canonical booking creation. |
| `GET /bookings/:id` | Partial | Detail route exists. |
| `POST /bookings` | Scaffold | Route exists but ignores real selected slot and creates an artificial booking time. Do not build new UX against this behavior. |
| `POST /bookings/:id/cancel` | Partial | Cancel exists and refund math exists, but broader lifecycle is incomplete. |
| `POST /payments/create-intent` | Scaffold | Still returns stubbed values. |
| `POST /stripe/webhook` | Partial | Handles Stripe signature parsing and `account.updated`, but durable idempotency/async processing is still missing. |

### Bonus / Admin

| Endpoint | Status | Notes |
|---|---|---|
| `GET /bonuses/current` | Live | Returns current or latest period snapshot. |
| `GET /bonuses/leaderboard` | Partial | Real route exists, but period/award logic is not yet authoritative. |
| `GET /admin/dashboard` | Partial | Dashboard exists, but stats surface is still basic. |
| `GET /admin/users` | Partial | User list exists. |
| `POST /admin/users/:id/suspend` | Partial | Suspend action exists; downstream enforcement is still incomplete. |
| `GET /admin/disputes` | Partial | Dispute list exists. |
| `POST /admin/disputes/:id/resolve` | Partial | Resolve action exists, but refund/re-clean execution is not complete. |

---

## Canonical MVP Target Surface

These are the interfaces new work should build toward.

| Endpoint | Status | Purpose |
|---|---|---|
| `GET /search/cleaners?lat=&lng=&date=` | Partial | Must honor `date` and exclude cleaners without valid availability. |
| `GET /cleaners/:id/availability?date=&duration=` | Live | Canonical slot source for customer booking UI. |
| `POST /booking/holds` | Planned | Create TTL-bound booking hold with idempotency and conflict checks. |
| `POST /payments/create-intent` | Scaffold | Must become real Stripe-backed intent creation or be folded into confirm flow. |
| `POST /booking/confirm` | Planned | Turn a valid hold plus successful payment into a confirmed booking. |
| `GET /me/bookings` | Partial | Should represent canonical booking lifecycle after hold/confirm is in place. |
| `POST /bookings/:id/cancel` | Partial | Needs to operate against the canonical lifecycle with refund side effects. |
| `POST /bookings/:id/complete` | Planned | Cleaner marks work complete; starts confirm/auto-confirm path. |
| `POST /bookings/:id/review` | Planned | Customer submits one review for one completed booking. |
| `POST /bookings/:id/dispute` | Planned | Customer opens dispute against a completed booking. |

---

## Key Endpoint Guidance

### `GET /search/cleaners`

**Current live behavior**

- Supports `lat`, `lng`, `radiusMeters`, `minRating`, `maxRate`, `services`, `sortBy`, and `limit`.
- Returns public cleaner cards with rate, rating, services, and distance.
- Does **not** currently enforce `date` availability filtering.

**Canonical direction**

- `date` must be honored.
- `maxRate` should be treated consistently as integer cents across contract, backend, and web client.
- Cursor pagination should be added if this becomes a launch or scale requirement.

### `GET /cleaners/:id/availability`

**Current live behavior**

Request:

```http
GET /cleaners/{id}/availability?date=2026-03-10&duration=3
```

Response:

```json
{
  "date": "2026-03-10",
  "slots": [
    {
      "start": "2026-03-10T17:00:00.000Z",
      "end": "2026-03-10T20:00:00.000Z",
      "available": true
    }
  ]
}
```

**Canonical direction**

- Customer booking UI should render these returned slots directly.
- Do not keep hardcoded time-slot lists once the profile/booking flow is rebuilt.

### `POST /bookings`

**Current live behavior**

Request shape today:

```json
{
  "cleanerId": "uuid",
  "slotId": "string",
  "duration": 3,
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102",
    "unit": "4B"
  }
}
```

**Important note**

- This route currently ignores the requested slot and creates a booking at an artificial future time.
- It should be treated as a transitional endpoint only.

### `POST /payments/create-intent`

**Current live behavior**

```json
{
  "clientSecret": "pi_stub_..._secret",
  "paymentIntentId": "pi_stub_..."
}
```

This is placeholder behavior and should not be treated as a production-ready payment API.

### `POST /stripe/webhook`

**Current live behavior**

- Verifies Stripe signature.
- Handles `account.updated` for Connect onboarding state.
- Deduplicates using in-memory state only.

**Canonical direction**

- Persist event receipt in the database.
- Process asynchronously with retries and DLQ handling.
- Support payment and payout event handling in addition to Connect onboarding.

---

## Booking Lifecycle Contract

### Canonical state progression

```text
availability slot
-> hold created
-> payment intent created
-> booking confirmed
-> cleaner completes
-> customer confirms or auto-confirms
-> payout released
-> review or dispute
```

### Booking statuses

- `confirmed`
- `in_progress`
- `completed`
- `cancelled_by_customer`
- `cancelled_by_cleaner`
- `disputed`

These statuses already exist in the shared contract and database schema. New endpoints should use them consistently instead of inventing new variants.

---

## Idempotency

Canonical endpoints that should require `X-Idempotency-Key`:

- `POST /auth/sync`
- `POST /booking/holds`
- `POST /payments/create-intent`
- `POST /booking/confirm`
- `POST /bookings/:id/cancel`
- `POST /bookings/:id/complete`
- `POST /bookings/:id/review`
- `POST /bookings/:id/dispute`
- admin refund-resolution actions

Current storage model already available in the schema:

- `booking_holds.idempotency_key`
- `api_idempotency_keys`
- `webhook_events`

What is still missing is consistent use of those tables across the live handlers.

---

## Versioning

Current live backend routes are mostly unversioned.

Canonical public contract should normalize around:

- `/v1` for current stable surface
- `/v2` only when a true breaking change is needed

Do not add new divergent route shapes casually while Track 0 is still open.

---

## Build Priorities

If you are building from this spec, use this order:

1. Finish `Track 0` in [PLAN.md](./PLAN.md)
2. Make search and profile use real availability end to end
3. Implement `POST /booking/holds`
4. Replace stub payment intent behavior
5. Implement `POST /booking/confirm`
6. Implement completion, payout, review, and dispute lifecycle

---

## Related Docs

- [PLAN.md](./PLAN.md)
- [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)
- [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md)
- [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md)
- [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md)
