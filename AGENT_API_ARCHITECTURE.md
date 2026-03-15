# Agent API Architecture (MCP Server)

## Overview

Sweep exposes an MCP (Model Context Protocol) server so AI agents — Claude, ChatGPT, Cursor, or any MCP-compatible client — can act on behalf of a user: searching for cleaners, creating bookings, and processing payments. The agent never sees raw credentials or card data. Every action is scoped, audited, and subject to user-controlled approval gates.

**Target audience:** Tech-savvy customers who use AI agents as their daily interface. An agent says "book me a cleaner for Tuesday" and Sweep handles it end-to-end.

**Reference docs:** [API_SPECIFICATION.md](./API_SPECIFICATION.md) | [BACKEND_ARCHITECTURE.md](./BACKEND_ARCHITECTURE.md) | [SYSTEM_DESIGN_OVERVIEW.md](./SYSTEM_DESIGN_OVERVIEW.md)

---

## Architecture

```
AI Agent (Claude, ChatGPT, etc.)
    │
    │  Streamable HTTP + OAuth 2.1 Bearer Token
    │
    ▼
┌──────────────────────────────────┐
│  /.well-known/oauth-protected-   │  RFC 9728 metadata
│  resource                        │  → points to Clerk
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│  Clerk (Authorization Server)    │
│  ─ OAuth 2.1 + PKCE (S256)       │
│  ─ Consent screen                │
│  ─ Registered clients at launch  │
│  ─ Opaque tokens preferred       │
└──────────────────────────────────┘
    │
    │  Access Token (Clerk-managed lifetime)
    ▼
┌──────────────────────────────────┐
│  MCP Gateway (Elysia middleware) │
│  ─ Origin validation             │
│  ─ Token verification (Clerk)    │
│  ─ Resource/audience checks      │
│  ─ Capability + spend checks     │
│  ─ Audit logging (OpenTelemetry) │
│  ─ Abuse / loop detection        │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│  Sweep MCP Server                │
│  POST /mcp  (Streamable HTTP)    │
│  ─ JSON-RPC 2.0 tool dispatch    │
│  ─ Session management            │
│  ─ Input validation (Zod)        │
│  ─ Tiered approval routing       │
│  ─ Content sanitization          │
└──────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────┐
│  Existing Backend Services       │
│  ─ Prisma / PostgreSQL (PostGIS) │
│  ─ Stripe Connect (SetupIntents, │
│    PaymentIntents, payouts)      │
│  ─ Optional SPT adapter later    │
│  ─ Redis (sessions, denylist)    │
│  ─ Notification service (push)   │
└──────────────────────────────────┘
```

### Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **Streamable HTTP transport** (not stdio) | Cloud-hosted server; agents connect over the internet; supports OAuth, sessions, and concurrency |
| **Clerk as OAuth Authorization Server** | Already our auth provider; has `@clerk/mcp-tools` helpers, consent screens, and MCP metadata support |
| **Standard Clerk OAuth scopes + Sweep capabilities** | Clerk does not yet support custom OAuth scopes; use OIDC scopes for identity and enforce Sweep-specific permissions in our own DB |
| **Gateway as Elysia middleware** (not a separate service) | Keeps modular monolith pattern; avoids adding another deployment target at MVP |
| **Saved payment method flow for MVP** | Works with today's Sweep checkout architecture, does not depend on Stripe preview access, and still keeps card data away from agents |
| **SPT adapter only behind a feature flag** | Stripe Agentic Commerce/SPTs are still private preview; add later only for approved partners and supported agents |
| **Registered OAuth clients at launch** | Safer default than open dynamic registration; evaluate DCR later if a target MCP client truly requires it |
| **Tools wrap existing API endpoints** (not new data paths) | MCP tools call the same domain logic as web/mobile; single source of truth |

---

## Transport: Streamable HTTP

The MCP server uses a single MCP endpoint path: `/mcp`

| Direction | Mechanism |
|-----------|-----------|
| Client → Server | HTTP POST with JSON-RPC 2.0 body |
| Server → Client | JSON (`application/json`) or SSE stream (`text/event-stream`) |
| Optional streaming | Client can open `GET /mcp` when SSE streaming is enabled |
| Session tracking | `Mcp-Session-Id` header assigned on `initialize`, required on all subsequent requests |

Sessions are stored in Redis, keyed by `<user_id>:<session_id>`. Sessions expire after 2 hours of inactivity.

Transport hardening:

- Validate the `Origin` header on every `GET /mcp` and `POST /mcp`; invalid origins return `403`
- Require HTTPS in production
- Validate `Mcp-Session-Id` format and bind it to the authenticated user + agent client
- Keep the legacy `/.well-known/oauth-authorization-server` endpoint available for older MCP clients when needed

---

## Authentication & Authorization

### OAuth 2.1 Flow

```
1. Agent connects to `POST /mcp` without a token
2. Server returns 401 + WWW-Authenticate pointing to resource metadata
3. Agent fetches `GET /.well-known/oauth-protected-resource/mcp`
   → discovers Clerk as the authorization server
4. Agent initiates Authorization Code + PKCE (S256) flow with Clerk
5. User sees Clerk consent screen for standard OIDC scopes (`openid profile email`)
6. Sweep shows or stores a first-party grant defining the agent's allowed capabilities
7. Clerk issues an OAuth access token and refresh token using Clerk-managed lifetimes
8. Agent calls `POST /mcp` with `Authorization: Bearer <token>`
9. Gateway verifies the token with Clerk, validates issuer + resource/audience binding, then loads the matching Sweep grant and capabilities
```

### Client Registration Strategy

Launch strategy:

- Register and review supported MCP clients in Clerk ahead of time
- Keep the Clerk consent screen enabled for every OAuth app
- Show the exact agent name and granted capabilities in Sweep settings

Dynamic Client Registration (RFC 7591) is not enabled by default at launch. It stays behind a later hardening milestone because it creates a public unauthenticated registration endpoint and increases phishing and abuse risk.

### Clerk OAuth Scopes

Clerk currently supports standard OAuth/OIDC scopes such as:

- `openid`
- `profile`
- `email`
- `public_metadata`
- `private_metadata`

For the MCP server, the protected resource metadata advertises only the minimum identity scopes needed to identify the user and show a meaningful consent screen:

- `openid`
- `profile`
- `email`

### Sweep Capabilities

Sweep-specific permissions are enforced as first-party capabilities stored in Sweep's database, not as Clerk OAuth scopes.

| Capability | Grants access to |
|------------|------------------|
| `marketplace.read` | `search_cleaners`, `get_cleaner_profile`, `check_availability`, `get_booking_quote` |
| `bookings.read` | `get_my_bookings` |
| `bookings.create` | `create_booking` |
| `bookings.manage` | `cancel_booking`, `reschedule_booking` |
| `payments.charge` | `confirm_and_pay_booking` |
| `reviews.write` | `rate_cleaner` |

If an agent attempts an action without the matching capability, the server returns `403` with a structured error such as:

```json
{
  "error": {
    "code": "AGENT_CAPABILITY_REQUIRED",
    "requiredCapability": "payments.charge"
  }
}
```

### Token Lifecycle

| Token | Lifetime | Storage |
|-------|----------|---------|
| Access token | Clerk-managed (currently 1 day by default) | Agent memory |
| Refresh token | Clerk-managed | Secure agent storage |
| MCP session | 2 hours idle timeout | Redis |

Recommended token format: **opaque access tokens** for instant revocation. Clerk also supports JWT access tokens, but if Sweep chooses JWTs later we must keep local revocation checks because JWTs remain valid until expiry.

### Revocation

- User revokes from web/mobile settings → mark the local agent grant revoked, clear active MCP sessions, and deny future tool calls immediately
- If opaque tokens are enabled, revoke the Clerk OAuth grant/token as well
- Gateway checks local grant status on every request
- "Revoke all agents" panic button clears every active agent grant and session for the user

---

## MCP Tool Catalog

### Read-Only Tools (capabilities: `marketplace.read`, `bookings.read`)

#### `search_cleaners`
Search for available cleaners by location, date, and filters.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| lat | number | yes | Latitude |
| lng | number | yes | Longitude |
| date | ISO 8601 | no | Filter by availability |
| min_rating | number | no | Minimum rating (1-5) |
| max_rate | number | no | Maximum hourly rate (cents) |
| services | string[] | no | Required services |
| sort_by | string | no | `distance`, `rating`, `price` |

Returns: list of cleaners with name, rate, rating, distance, next available slot. User-generated content (bios) is sanitized before returning.

#### `get_cleaner_profile`
Full public profile for a specific cleaner.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| cleaner_id | uuid | yes | Cleaner ID |

Returns: bio, rate, services, badges, rating, review count, availability summary.

#### `check_availability`
Check a specific cleaner's open time slots.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| cleaner_id | uuid | yes | Cleaner ID |
| date | ISO 8601 | yes | Date to check |
| duration | number | no | Hours needed (default: 3) |

Returns: list of available time slots for the requested date.

#### `get_booking_quote`
Calculate the price for a specific booking configuration.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| cleaner_id | uuid | yes | Cleaner ID |
| start_time | ISO 8601 | yes | Booking start |
| end_time | ISO 8601 | yes | Booking end |
| services | string[] | yes | Requested services |

Returns: `{ subtotal, service_addons, platform_fee, total }`. All amounts server-calculated.

#### `get_my_bookings`
List the authenticated user's bookings.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| status | string | no | Filter: `confirmed`, `in_progress`, `completed`, `cancelled_by_customer`, `cancelled_by_cleaner`, `disputed` |
| limit | number | no | Results per page (default: 20) |

Returns: paginated booking list with cleaner info, times, and amounts.

### Mutation Tools

#### `create_booking` (capability: `bookings.create`)
Create a booking hold. Subject to tiered approval.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| cleaner_id | uuid | yes | Cleaner to book |
| start_time | ISO 8601 | yes | Booking start |
| end_time | ISO 8601 | yes | Booking end |
| services | string[] | yes | Requested services |
| address | object | yes | Service address |
| idempotency_key | uuid | yes | Prevents duplicate bookings |

Returns: booking hold with pricing breakdown and `approval_status` (`approved`, `pending_user_approval`).

Behavior:

- **Tier 2 / auto-approved**: create a normal 5-minute booking hold immediately
- **Tier 3 / approval required**: do **not** create the normal hold yet; instead store a quote snapshot in `agent_pending_approvals`, notify the user, and return `{ approval_status: "pending_user_approval", approval_id, expires_at }`
- When the user approves, the server revalidates availability and pricing, then atomically creates the standard 5-minute booking hold
- If the slot is no longer available or the total has changed above the approved amount, return a structured `needs_requote` or `booking_conflict` response rather than silently continuing

#### `confirm_and_pay_booking` (capability: `payments.charge`)
Confirm a held booking and process payment.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| hold_id | uuid | yes | Booking hold ID |
| idempotency_key | uuid | yes | Prevents duplicate payments |

The server:
1. Recalculates the amount server-side (never trusts agent-provided amounts)
2. Loads the customer's saved Stripe payment method (existing Sweep flow)
3. Creates an off-session PaymentIntent with `application_fee_amount` = 8% platform fee and `transfer_data.destination` = cleaner's Connect account
4. If Stripe returns `requires_action` or no reusable payment method exists, return `requires_customer_payment` and hand off to web/mobile checkout
5. On success, transitions booking to `confirmed`

Returns: confirmed booking details or structured error.

Optional future adapter:

- If Sweep gets access to Stripe Agentic Commerce preview and a target agent can provide a granted `SharedPaymentToken`, add an optional `shared_payment_granted_token` parameter behind a feature flag
- In that mode, the agent provisions the token and Sweep uses it when creating the `PaymentIntent`

#### `cancel_booking` (capability: `bookings.manage`)
Cancel an existing booking.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| booking_id | uuid | yes | Booking to cancel |
| reason | string | no | Cancellation reason |

Returns: cancellation confirmation with refund amount (if applicable).

#### `reschedule_booking` (capability: `bookings.manage`)
Change the date/time of an existing booking.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| booking_id | uuid | yes | Booking to reschedule |
| new_start_time | ISO 8601 | yes | New start time |
| new_end_time | ISO 8601 | yes | New end time |
| idempotency_key | uuid | yes | Prevents duplicate reschedules |

Returns: updated booking with new pricing (if duration changed).

#### `rate_cleaner` (capability: `reviews.write`)
Submit a rating and review for a completed booking.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| booking_id | uuid | yes | Completed booking |
| rating | number | yes | 1-5 stars |
| text | string | no | Review text |

Returns: created review.

---

## Tiered Approval Model

Users control what agents can do autonomously vs. what requires explicit approval.

| Tier | Actions | Approval mechanism | Default |
|------|---------|-------------------|---------|
| **0 — Read** | Browse cleaners, check prices, view availability | Blanket consent at OAuth authorization | Always on |
| **1 — Low-risk** | View own bookings, save preferences | Blanket consent with audit log | Always on |
| **2 — Moderate** | Create hold and pay within spending cap for a previously used cleaner | Pre-authorized, auto-approved | Cap: $150/booking |
| **3 — High-value** | Book above cap, use a first-time cleaner, or proceed after quote drift | Push notification → approve/deny (15 min TTL) | Requires approval |
| **4 — Sensitive** | Future actions such as changing payment method, recurring bookings, refunds | In-app confirmation + Clerk reverification | Not in Slice 7 MVP |

Definition: **first-time cleaner** means the authenticated customer has no prior confirmed or completed booking with that cleaner.

### Spending Limits (User-Configurable)

| Limit | Default | Description |
|-------|---------|-------------|
| Per-transaction | $150 | Max single booking amount |
| Daily aggregate | $300 | Max total agent-initiated spend per day |
| Weekly aggregate | $500 | Rolling 7-day max |

Limits are stored in a dedicated `agent_spending_limits` table. The gateway checks projected totals during `create_booking` and re-checks them during `confirm_and_pay_booking`. If exceeded, the tool returns a structured `pending_user_approval` response instead of failing.

### Approval Flow

```
Agent calls create_booking($250 booking)
    │
    ▼
Gateway checks: $250 > $150 per-transaction cap → Tier 3
    │
    ▼
Server creates pending approval with quote snapshot
    │
    ▼
Push notification sent to user's device:
  "Agent wants to book Maria's Cleaning for $250 on Tue Mar 18, 2-5pm. Approve?"
    │
    ├─ User taps Approve → server revalidates slot + price → booking hold created if still valid
    ├─ User taps Deny → agent receives { status: "denied", reason: "user_rejected" }
    └─ 15 min timeout → agent receives { status: "expired" }
```

---

## Payment Flow (Agent-Initiated, MVP)

MVP agent payments use Sweep's existing Stripe saved-payment-method flow. The agent never sees raw card data.

### Prerequisites

- User has a saved payment method (Stripe `SetupIntent` with `usage: off_session` created during normal web/mobile onboarding)
- User has granted the `payments.charge` capability to the agent
- User has passed any Tier 3 approval required for the hold amount or cleaner selection

### Flow

```
1. Agent calls confirm_and_pay_booking(hold_id)
2. Server recalculates amount from hold record (NEVER uses agent-provided amount)
3. Server checks the approved amount ceiling and current spending limits again
4. Server creates PaymentIntent using the saved customer payment method:
   - amount: booking total
   - application_fee_amount: 8% platform fee (PLATFORM_FEE_BPS = 800)
   - transfer_data.destination: cleaner's Stripe Connect account
   - confirm: true
   - off_session: true
5. If Stripe returns `requires_action`:
   - Return `requires_customer_payment`
   - Deep-link the user into the normal Sweep checkout UI
6. On payment_intent.succeeded webhook:
   - Booking transitions to confirmed
   - Notification sent to cleaner
   - Audit record created
7. Agent receives confirmed booking response
```

### Optional Future SPT Adapter

Stripe Shared Payment Tokens are a good fit for some external-agent ecosystems, but they are still in private preview and the agent, not Sweep, provisions the token. If Sweep later enables this path:

1. Agent provisions and grants the SPT to Sweep or the connected account
2. Agent passes the granted token reference to `confirm_and_pay_booking`
3. Sweep creates the `PaymentIntent` using that granted token
4. The path remains behind a feature flag until preview access, webhook handling, and partner testing are complete

### PCI Compliance

Card data still flows exclusively through Stripe. Agents never receive raw payment credentials. Because the MVP path uses Sweep's existing Stripe integration, the compliance posture should be reviewed as part of the normal payments launch checklist rather than assuming a new SAQ classification in this doc.

---

## Security

### Threat Model

| Threat | Severity | Mitigation |
|--------|----------|------------|
| **Prompt injection** — malicious cleaner bio manipulates agent into unauthorized booking | Critical | Sanitize all user-generated content before returning to agents. Server-side enforcement of all authorization — the LLM proposes, the server validates. Typed Zod schemas for all inputs/outputs. |
| **Token theft / replay** | High | Prefer opaque tokens for instant revocation. TLS 1.3 only. Never log full tokens (hash prefix only). Local grant revocation checked on every request. |
| **Tool chaining escalation** — combining low-privilege tools to bypass limits | High | Aggregate spending limits enforced server-side. Velocity checks flag unusual patterns (10 bookings in 5 min). Capability validation on every tool call. |
| **Runaway agent loops** | Medium | Loop detection: if same tool called with same params >5 times/min, return error. Per-session rate limits. Circuit breaker on repeated failures. |
| **Cross-tenant data leak** | Critical | Every tool call validates that the requested resource belongs to the authenticated user. Booking lookups filtered by user_id. |
| **DNS rebinding / browser-origin abuse** | High | Validate `Origin` on all MCP HTTP requests. Reject invalid origins with `403`. |
| **Approval abuse / slot squatting** | High | Tier 3 requests store approval snapshots, not long-lived holds. Availability is revalidated on approval. Rate limit approval creation per user and per cleaner slot. |

### Input Validation

Every tool parameter is validated via Zod schemas from `@sweep/shared`. The MCP server rejects malformed input with structured error messages before hitting the backend.

```
Origin validation → Clerk token verification → Resource/audience check → Zod validation → Capability check → Spending limit check → Backend call
```

### Content Sanitization

User-generated content returned to agents (cleaner bios, review text, booking notes) is stripped of:
- HTML/script tags
- Common prompt injection patterns (e.g., "ignore previous instructions")
- Control characters

This is defense-in-depth — the real security boundary is server-side enforcement, not LLM behavior.

---

## Rate Limiting

| Limit | Value | Scope |
|-------|-------|-------|
| Read tool calls | 60/min | Per user |
| Write tool calls | 10/min | Per user |
| Payment tool calls | 5/min | Per user |
| Total calls per session | 500 | Per session |

Enforcement via Redis sliding window counters. Response on limit exceeded:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "retryAfter": 30
  }
}
```

Progressive response: warn at 80% of limit via response header, soft-block at 100%, hard-block with cooldown at 150%.

---

## Audit Logging

Every agent-initiated action produces an immutable audit record:

| Field | Description |
|-------|-------------|
| `user_id` | Authenticated user |
| `agent_client_id` | Agent platform identity (from OAuth client) |
| `session_id` | MCP session ID |
| `tool_name` | Tool invoked |
| `tool_params` | Input parameters (PII redacted) |
| `capability_used` | Sweep capability that authorized this call |
| `approval_tier` | Which tier applied (0-4) |
| `approval_required` | Whether user approval was needed |
| `approval_result` | `auto`, `approved`, `denied`, `expired` |
| `result_status` | Success or error code |
| `amount` | For financial actions: amount in cents |
| `booking_id` | For booking actions: booking reference |
| `timestamp` | ISO 8601 with timezone |

Records are append-only, stored in a dedicated `agent_audit_logs` table. Retained for 2 years minimum (compliance).

---

## Database Changes

New tables required:

### `agent_authorizations`
Durable per-user grants for a specific agent platform.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| user_id | uuid | FK → users |
| agent_client_id | text | OAuth client ID of the agent platform |
| agent_name | text | Display name shown to the user |
| clerk_scopes | text[] | Granted Clerk OAuth scopes |
| capabilities | text[] | Sweep permissions granted to this agent |
| status | enum | `active`, `revoked` |
| consented_at | timestamptz | When the user granted access |
| revoked_at | timestamptz | Null unless revoked |
| updated_at | timestamptz | Last capability change |

Unique index on `(user_id, agent_client_id)`.

### `agent_sessions`
Tracks active MCP sessions.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| authorization_id | uuid | FK → agent_authorizations |
| user_id | uuid | FK → users |
| agent_client_id | text | OAuth client ID of the agent platform |
| mcp_session_id | text | MCP session identifier |
| token_format | text | `opaque` or `jwt` |
| created_at | timestamptz | Session start |
| last_active_at | timestamptz | Last tool call |
| revoked_at | timestamptz | Null unless revoked |

### `agent_spending_limits`
User-configurable spending caps for agent actions.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| user_id | uuid | FK → users (unique) |
| per_transaction_cents | int | Default: 15000 ($150) |
| daily_aggregate_cents | int | Default: 30000 ($300) |
| weekly_aggregate_cents | int | Default: 50000 ($500) |
| updated_at | timestamptz | Last modified |

### `agent_pending_approvals`
Pending Tier 3/4 approval requests.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| user_id | uuid | FK → users |
| session_id | uuid | FK → agent_sessions |
| tool_name | text | Tool that triggered approval |
| tool_params | jsonb | Serialized tool input |
| quote_snapshot | jsonb | Server-calculated pricing + cleaner + times |
| amount_cents | int | Amount requiring approval (if applicable) |
| status | enum | `pending`, `approved`, `denied`, `expired` |
| expires_at | timestamptz | 15-minute TTL |
| resolved_at | timestamptz | When user responded |
| created_at | timestamptz | When created |

### `agent_audit_logs`
Immutable audit trail for all agent actions.

| Column | Type | Description |
|--------|------|-------------|
| id | uuid | Primary key |
| user_id | uuid | FK → users |
| agent_client_id | text | Agent platform identity |
| session_id | uuid | FK → agent_sessions |
| tool_name | text | Tool invoked |
| tool_params | jsonb | Input (PII redacted) |
| capability_used | text | Sweep capability |
| approval_tier | int | 0-4 |
| approval_result | text | `auto`, `approved`, `denied`, `expired` |
| result_status | text | `success` or error code |
| amount_cents | int | For financial actions |
| booking_id | uuid | For booking actions |
| created_at | timestamptz | Timestamp |

Index on `(user_id, created_at)` for user-facing audit view.

---

## Frontend Changes

### Web: Agent Settings Page (`/settings/agents`)

- **Connected agents list**: shows agent name, platform, granted capabilities, last active
- **Spending limits**: configure per-transaction, daily, and weekly caps
- **Revoke access**: per-agent revoke button + "revoke all" panic button
- **Capability management**: toggle read, booking, management, payment, and review permissions
- **Audit log**: filterable list of agent actions with timestamps and outcomes

### Mobile: Agent Settings Screen

- Same capabilities as web
- Push notification handling for Tier 3/4 approval requests (approve/deny buttons)

### Web + Mobile: Approval Notifications

- Tier 3: push notification with booking details + approve/deny
- If price or availability changed during approval, show a fresh confirmation request instead of auto-continuing
- Tier 4: deep-link to in-app confirmation screen with full details + Clerk reverification

---

## Phased Rollout

### Phase 1: Read-Only (MVP)
- MCP endpoint with Streamable HTTP
- OAuth 2.1 via Clerk (PKCE, registered clients, consent)
- Read-only tools: `search_cleaners`, `get_cleaner_profile`, `check_availability`, `get_booking_quote`, `get_my_bookings`
- First-party capability grants
- Rate limiting and audit logging
- Agent settings page (web)

**Ship when:** agents can connect, authenticate, and browse the marketplace. No financial risk.

### Phase 2: Bookings
- Mutation tools: `create_booking`, `cancel_booking`, `reschedule_booking`, `rate_cleaner`
- Tiered approval system (Tiers 0-3)
- Spending limits (per-transaction and daily)
- Approval snapshot + revalidation flow for above-cap requests
- Push notification approval flow (mobile)

**Ship when:** agents can book cleaners with user-controlled guardrails.

### Phase 3: Payments
- `confirm_and_pay_booking` with Sweep's saved payment method flow
- Weekly aggregate spending limits
- `requires_customer_payment` handoff for SCA / missing-payment-method cases
- Full audit logging with financial context

**Ship when:** agents can complete the entire booking-to-payment flow.

### Phase 4: Hardening
- MCP gateway observability (OpenTelemetry traces for every tool call)
- Abuse detection (anomaly scoring on agent behavior)
- Loop detection and circuit breakers
- Evaluate DPoP / token binding when supported by Clerk
- Evaluate dynamic client registration only after risk review
- Optional Stripe SPT adapter for preview-enabled partners
- Canary rollout: 10% → 50% → 100% of users

**Ship when:** production-ready with full observability and abuse prevention.

---

## Dependencies

| Dependency | Version / Status | Notes |
|------------|-----------------|-------|
| MCP Specification | 2025-11-25 | Current Streamable HTTP + authorization guidance |
| `@clerk/mcp-tools` | SDK available | Handles protected resource metadata and OAuth token verification |
| Stripe Agentic Commerce / SPTs | Private preview | Optional future adapter only; not required for Slice 7 GA |
| Expo Push Notifications | Required for Tier 3/4 approvals | Already in mobile stack |

---

## Open Questions

1. **Client onboarding:** Do we launch with an allowlist of reviewed clients only, or accept the phishing/cleanup tradeoffs of dynamic client registration sooner?
2. **Approval drift policy:** If a Tier 3 approval revalidation changes the price or loses the slot, do we auto-expire or let the user approve the new quote in the same flow?
3. **Sensitive-action reverification:** Which future actions should require Clerk reverification instead of a simple push approval?
4. **External-agent payments:** Do we pursue Stripe's SPT preview for partner agents later, or stay with Sweep-saved payment methods only?
5. **Recurring bookings / cleaner-side MCP:** Are either of these in scope for post-MVP agent work, or explicitly out of scope for now?
