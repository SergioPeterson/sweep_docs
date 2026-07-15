# API Specification

## Overview

This document now tracks two things at once:

1. the **live backend surface that exists today**
2. the **canonical MVP contract** the team should build toward next

The earlier version of this document described a mostly future-state API and incorrectly said the backend only exposed scaffold endpoints. That is no longer accurate.

### Status As Of July 15, 2026

- The backend has real onboarding, search/browse, booking, payments, payouts, reviews, bonus, admin, and MCP surfaces.
- The customer booking flow is canonical: availability, hold, payment intent, then confirmation.
- The remaining launch work is production provisioning and operational verification, not placeholder booking or payment handlers.

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
- Booking prices, payments, payouts, refunds, and bonus amounts are integer cents at the API boundary.
- `GET /search/cleaners.maxRate` is also integer cents. The web and mobile clients convert their dollar UI values before issuing the request.

### Idempotency

Use `X-Idempotency-Key` on canonical mutating booking/payment/dispute endpoints.

### Canonical Booking Flow

The canonical customer path is:

`GET /cleaners/:id/availability -> POST /booking/holds -> POST /payments/create-intent -> POST /booking/confirm`

`POST /bookings` is an admin-only compatibility endpoint.
It validates real availability and conflicts, but bypasses payment collection, so customer clients must not use it.

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
| `GET /search/cleaners` | Live | Supports date and duration filtering against real availability. Cursor pagination is still a scale follow-up. |
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
| `POST /cleaner/stripe-connect` | Live | Creates Stripe Connect onboarding links and persists webhook-driven onboarding state. |
| `GET /cleaner/dashboard` | Live | Returns real cleaner booking and operational data. |
| `GET /cleaner/earnings` | Live | Returns payout-backed earnings data. |
| `GET /cleaner/bonuses/summary` | Live | Returns the cleaner's calculated bonus summary. |

### Booking / Payment

| Endpoint | Status | Notes |
|---|---|---|
| `POST /booking/holds` | Live | Creates TTL-bound, idempotent holds with pricing snapshots and slot-conflict checks. |
| `POST /payments/create-intent` | Live | Creates a Stripe PaymentIntent for a valid hold. |
| `POST /booking/confirm` | Live | Confirms a valid held booking only after successful payment. |
| `GET /me/bookings` | Live | Returns customer booking history for the canonical lifecycle. |
| `GET /bookings/:id` | Live | Returns booking detail with ownership checks. |
| `POST /bookings` | Partial | Admin-only compatibility creation path. It validates live availability but bypasses customer payment collection. |
| `POST /bookings/:id/cancel` | Live | Applies the refund policy and records the outcome. |
| `POST /bookings/:id/complete` | Live | Completes work and initiates payout processing. |
| `POST /reviews` | Live | Creates one customer review per completed booking. |
| `POST /bookings/:id/dispute` | Live | Opens a customer dispute with ownership and state guards. |
| `POST /stripe/webhook` | Live | Persists idempotent webhook receipt and processes payment, payout, and Connect events with retry and dead-letter handling. |

### Bonus / Admin

| Endpoint | Status | Notes |
|---|---|---|
| `GET /bonuses/current` | Live | Returns current or latest period snapshot. |
| `GET /bonuses/leaderboard` | Live | Returns bonus-period leaderboard data. |
| `GET /admin/dashboard` | Live | Returns date-filtered operational metrics. |
| `GET /admin/users` | Live | Lists users for operations. |
| `POST /admin/users/:id/suspend` | Live | Suspension is enforced in customer booking and cleaner search paths. |
| `GET /admin/disputes` | Live | Lists disputes for operational triage. |
| `POST /admin/disputes/:id/resolve` | Live | Resolves disputes and executes full or partial Stripe refunds. |
| `GET /admin/payouts` | Live | Lists payouts, including failed payouts for reconciliation. |
| `POST /admin/payouts/:id/retry` | Live | Retries failed Stripe transfers with idempotency. |

---

## Canonical MVP Target Surface

These are the interfaces new work should build toward.

| Endpoint | Status | Purpose |
|---|---|---|
| `GET /search/cleaners?lat=&lng=&date=&duration=` | Live | Filters cleaners by real availability for the requested date and duration. |
| `GET /cleaners/:id/availability?date=&duration=` | Live | Canonical slot source for customer booking UI. |
| `POST /booking/holds` | Live | Creates a TTL-bound booking hold with idempotency, pricing snapshot, and conflict checks. |
| `POST /payments/create-intent` | Live | Creates a Stripe-backed PaymentIntent for the hold. |
| `POST /booking/confirm` | Live | Turns a valid hold plus a successful payment into a confirmed booking. |
| `GET /me/bookings` | Live | Represents the canonical booking lifecycle. |
| `POST /bookings/:id/cancel` | Live | Cancels against the canonical lifecycle and records refund outcomes. |
| `POST /bookings/:id/complete` | Live | Marks cleaner completion and starts customer-confirm or auto-confirm processing. |
| `POST /reviews` | Live | Customer submits one review for one completed booking. |
| `POST /bookings/:id/dispute` | Live | Customer opens a dispute against a completed booking. |

---

## Key Endpoint Guidance

### `GET /search/cleaners`

**Current live behavior**

- Supports `lat`, `lng`, `radiusMeters`, `minRating`, `maxRate`, `services`, `sortBy`, `limit`, `date`, and `duration`.
- Returns public cleaner cards with rate, rating, services, and distance.
- Applies date and duration filtering using the same availability, blackout, booking, and active-hold constraints as booking creation.

**Canonical direction**

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

- Customer booking UI renders these returned slots directly.

### `POST /bookings`

**Current live behavior - admin compatibility only**

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

- This route accepts `startTime` and `endTime`, validates the cleaner's local availability, blackouts, existing bookings, and active holds, then creates a confirmed booking.
- It bypasses PaymentIntent collection and is restricted to admins.
- Customer clients must use the hold, payment intent, and confirm flow.

### `POST /payments/create-intent`

**Current live behavior**

```json
{
  "holdId": "uuid",
  "paymentMethodId": "pm_optional",
  "savePaymentMethod": true
}
```

The response contains the Stripe `clientSecret`, `paymentIntentId`, amount in cents, and currency.
The client confirms payment with Stripe, then calls `POST /booking/confirm`.

### `POST /stripe/webhook`

**Current live behavior**

- Verifies Stripe signature.
- Persists receipt in `webhook_events` before processing.
- Handles Connect onboarding, successful payment intents, and payout-related state.
- Retries failed or stale work and dead-letters exhausted attempts.

**Canonical direction**

- Configure production alerting, rollback, and runbook ownership around the existing durable processing path.

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

The live booking, payment, and webhook handlers use these idempotency records.

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

1. Provision production credentials and infrastructure for Stripe, Resend, S3, and observability.
2. Confirm branch protection, required checks, Code Owner review, dependency review, and provenance enforcement in GitHub.
3. Run production-like browser and operational smoke checks during rollout.

---

## Related Docs

- [PLAN.md](./PLAN.md)
- [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md)
- [FRONTEND_ARCHITECTURE.md](./FRONTEND_ARCHITECTURE.md)
- [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md)
- [IMPLEMENTATION_GUARDRAILS.md](./IMPLEMENTATION_GUARDRAILS.md)
