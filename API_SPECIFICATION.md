# API Specification

## API Overview

**Base URL:** `https://api.yourdomain.com/v1`

**Protocol:** REST over HTTPS
**Data Format:** JSON
**Authentication:** Clerk JWT (Bearer tokens)
**Rate Limiting:** 100 requests/minute per user, 1000/minute per IP

**Implementation Status (February 9, 2026):** This document defines the target MVP API contract. The current backend scaffold only exposes `GET /health`, `GET /`, and `GET /api/v1/status`.

---

## Authentication

### JWT Token Format (Clerk-issued)

**Session Token (example claims):**
```json
{
  "iss": "https://clerk.yourdomain.com",
  "sub": "user_2abcxyz",
  "sid": "sess_2abcxyz",
  "iat": 1739011200,
  "exp": 1739012100
}
```

Token refresh is handled by Clerk sessions. The platform API validates bearer tokens and maps `sub` to `users.id`/`users.external_auth_id`.

### Headers
```
Authorization: Bearer <access_token>
Content-Type: application/json
X-Idempotency-Key: <uuid>  # For idempotent mutating endpoints (see Idempotency section)
```

---

## Error Responses

### Standard Error Format
```json
{
  "error": {
    "code": "BOOKING_CONFLICT",
    "message": "This time slot is no longer available",
    "details": {
      "field": "startTime",
      "reason": "overlapping_booking"
    }
  },
  "timestamp": "2024-01-15T10:30:00Z",
  "path": "/v1/booking/holds"
}
```

### HTTP Status Codes
| Code | Meaning | When Used |
|------|---------|-----------|
| 200 | OK | Successful GET request |
| 201 | Created | Successful POST (resource created) |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing/invalid auth token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Double-booking, duplicate resource |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |
| 503 | Service Unavailable | Temporary outage |

---

## API Endpoints

### Authentication & Users

MVP authentication is managed by Clerk. The platform API expects a valid Clerk bearer token on protected routes and mirrors user metadata into the `users` table.

#### GET /auth/me
**Description:** Return current authenticated user profile and platform role

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "externalAuthId": "user_2abcxyz",
    "email": "user@example.com",
    "role": "customer",
    "emailVerified": true,
    "phoneVerified": true
  }
}
```

---

#### POST /auth/sync
**Description:** Idempotent upsert from Clerk identity into platform `users` row

**Headers:**
```
X-Idempotency-Key: <uuid>
```

**Request:**
```json
{
  "externalAuthId": "user_2abcxyz",
  "email": "user@example.com",
  "phone": "+14155551234",
  "firstName": "John",
  "lastName": "Doe",
  "role": "customer"
}
```

**Response (200):**
```json
{
  "user": {
    "id": "uuid",
    "externalAuthId": "user_2abcxyz",
    "role": "customer"
  }
}
```

---

### Cleaner Search & Discovery

#### GET /search/cleaners
**Description:** Search for cleaners by location and filters

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| lat | number | Yes | Latitude |
| lng | number | Yes | Longitude |
| radiusMeters | number | No | Search radius (default: 5000) |
| date | ISO 8601 | No | Filter by availability on date |
| minRating | number | No | Minimum rating (1-5) |
| maxRate | number | No | Maximum hourly rate (cents) |
| services | string[] | No | Required services (comma-separated) |
| sortBy | string | No | `distance`, `rating`, `price` (default: distance) |
| limit | number | No | Results per page (default: 20, max: 50) |
| cursor | string | No | Pagination cursor |

**Request:**
```
GET /search/cleaners?lat=37.7749&lng=-122.4194&radiusMeters=5000&minRating=4.5&sortBy=rating&limit=20
```

**Response (200):**
```json
{
  "cleaners": [
    {
      "id": "uuid",
      "firstName": "Jane",
      "lastName": "D.",
      "profileImageUrl": "https://cdn.../profile.jpg",
      "bio": "Professional cleaner with 5 years...",
      "baseRate": 5500,
      "minHours": 2,
      "rating": 4.8,
      "reviewCount": 42,
      "distance": 1250,
      "services": ["standard", "deep_clean", "inside_oven"],
      "badges": ["background_check", "insurance_verified"],
      "nextAvailable": "2024-01-20T09:00:00Z"
    }
  ],
  "meta": {
    "total": 45,
    "limit": 20,
    "cursor": "eyJpZCI6InV1aWQiLCJkaXN0YW5jZSI6MTI1MH0="
  }
}
```

---

#### GET /cleaners/:id
**Description:** Get detailed cleaner profile

**Response (200):**
```json
{
  "id": "uuid",
  "firstName": "Jane",
  "lastName": "Doe",
  "profileImageUrl": "https://cdn.../profile.jpg",
  "bio": "Professional cleaner with 5 years experience...",
  "baseRate": 5500,
  "minHours": 2,
  "radiusMeters": 8000,
  "rating": 4.8,
  "reviewCount": 42,
  "totalBookings": 120,
  "totalHours": 480,
  "repeatBookingRate": 65.5,
  "services": [
    {
      "type": "standard",
      "priceAddon": 0
    },
    {
      "type": "deep_clean",
      "priceAddon": 1500
    },
    {
      "type": "inside_oven",
      "priceAddon": 500
    }
  ],
  "badges": {
    "backgroundCheck": true,
    "insurance": true,
    "topRated": true
  },
  "memberSince": "2023-06-15T00:00:00Z",
  "responseTime": "< 1 hour",
  "cancellationRate": 2.5
}
```

---

#### GET /cleaners/:id/reviews
**Description:** Get reviews for a cleaner

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| limit | number | No | Results per page (default: 10) |
| cursor | string | No | Pagination cursor |

**Response (200):**
```json
{
  "reviews": [
    {
      "id": "uuid",
      "customer": {
        "firstName": "Alice",
        "profileImageUrl": "https://..."
      },
      "rating": 5,
      "text": "Jane did an amazing job! Very thorough and professional.",
      "response": "Thank you Alice! It was a pleasure working with you.",
      "services": ["deep_clean", "inside_oven"],
      "createdAt": "2024-01-10T15:30:00Z",
      "verified": true
    }
  ],
  "meta": {
    "total": 42,
    "averageRating": 4.8,
    "ratingDistribution": {
      "5": 35,
      "4": 5,
      "3": 2,
      "2": 0,
      "1": 0
    }
  }
}
```

---

#### GET /cleaners/:id/availability
**Description:** Get available time slots for a cleaner

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| date | ISO 8601 | Yes | Date to check (e.g., "2024-01-20") |
| duration | number | No | Required duration in hours (default: 3) |

**Request:**
```
GET /cleaners/uuid/availability?date=2024-01-20&duration=3
```

**Response (200):**
```json
{
  "date": "2024-01-20",
  "slots": [
    {
      "start": "2024-01-20T09:00:00Z",
      "end": "2024-01-20T12:00:00Z",
      "available": true
    },
    {
      "start": "2024-01-20T13:00:00Z",
      "end": "2024-01-20T16:00:00Z",
      "available": true
    }
  ]
}
```

---

### Booking

#### POST /booking/holds
**Description:** Create a temporary booking hold (5 min expiry)

**Headers:**
```
X-Idempotency-Key: <uuid>
```

**Request:**
```json
{
  "cleanerId": "uuid",
  "startTime": "2024-01-20T09:00:00Z",
  "endTime": "2024-01-20T12:00:00Z",
  "services": ["standard", "inside_oven"],
  "address": {
    "street": "123 Main St",
    "unit": "Apt 4B",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102",
    "accessInstructions": "Ring doorbell, code #4567"
  }
}
```

**Response (201):**
```json
{
  "hold": {
    "id": "uuid",
    "cleanerId": "uuid",
    "startTime": "2024-01-20T09:00:00Z",
    "endTime": "2024-01-20T12:00:00Z",
    "expiresAt": "2024-01-15T10:35:00Z",
    "pricing": {
      "baseRate": 5500,
      "hours": 3,
      "subtotal": 16500,
      "serviceAddons": 500,
      "platformFee": 1360,
      "total": 18360
    }
  }
}
```

**Errors:**
- 409 Conflict - Time slot no longer available
- 422 Unprocessable Entity - Invalid time range

---

#### POST /booking/confirm
**Description:** Confirm booking after payment success

**Headers:**
```
X-Idempotency-Key: <uuid>
```

**Request:**
```json
{
  "holdId": "uuid",
  "paymentIntentId": "pi_xxx"
}
```

**Response (201):**
```json
{
  "booking": {
    "id": "uuid",
    "status": "confirmed",
    "cleaner": {
      "id": "uuid",
      "firstName": "Jane",
      "profileImageUrl": "https://..."
    },
    "startTime": "2024-01-20T09:00:00Z",
    "endTime": "2024-01-20T12:00:00Z",
    "pricing": {
      "subtotal": 16500,
      "platformFee": 1360,
      "total": 18360
    },
    "services": ["standard", "inside_oven"],
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

---

#### POST /bookings
**Description:** Create a booking directly (MVP convenience endpoint that internally performs hold + confirm)

**Headers:**
```
X-Idempotency-Key: <uuid>
```

**Request:**
```json
{
  "cleanerId": "uuid",
  "slotId": "slot_uuid",
  "duration": 3,
  "address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102"
  }
}
```

**Response (201):**
```json
{
  "booking": {
    "id": "uuid",
    "status": "confirmed"
  }
}
```

---

#### GET /bookings/:id
**Description:** Get booking details

**Response (200):**
```json
{
  "id": "uuid",
  "status": "confirmed",
  "cleaner": {
    "id": "uuid",
    "firstName": "Jane",
    "lastName": "D.",
    "phone": "+14155551234",
    "profileImageUrl": "https://..."
  },
  "customer": {
    "id": "uuid",
    "firstName": "John",
    "phone": "+14155555678"
  },
  "startTime": "2024-01-20T09:00:00Z",
  "endTime": "2024-01-20T12:00:00Z",
  "pricing": {
    "subtotal": 16500,
    "platformFee": 1360,
    "total": 18360
  },
  "services": ["standard", "inside_oven"],
  "specialInstructions": "Please use eco-friendly products",
  "address": {
    "street": "123 Main St",
    "unit": "Apt 4B",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94102",
    "accessInstructions": "Ring doorbell, code #4567"
  },
  "createdAt": "2024-01-15T10:30:00Z"
}
```

---

#### POST /bookings/:id/cancel
**Description:** Cancel a booking

**Request:**
```json
{
  "reason": "Schedule conflict"
}
```

**Response (200):**
```json
{
  "booking": {
    "id": "uuid",
    "status": "cancelled_by_customer",
    "cancelledAt": "2024-01-16T14:00:00Z",
    "refundAmount": 18360,
    "refundReason": "Cancelled >24 hours in advance"
  }
}
```

---

#### POST /bookings/:id/complete
**Description:** Mark booking as complete (customer confirms)

**Response (200):**
```json
{
  "booking": {
    "id": "uuid",
    "status": "completed",
    "completedAt": "2024-01-20T12:00:00Z"
  }
}
```

---

#### POST /bookings/:id/dispute
**Description:** Create a dispute for a completed booking

**Headers:**
```
X-Idempotency-Key: <uuid>
```

**Request:**
```json
{
  "reason": "Service quality issue",
  "details": "Cleaner left before agreed end time"
}
```

**Response (201):**
```json
{
  "dispute": {
    "id": "uuid",
    "bookingId": "uuid",
    "status": "open",
    "createdAt": "2024-01-20T15:00:00Z"
  }
}
```

---

#### GET /me/bookings
**Description:** Get current user's bookings

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| status | string | No | Filter by status (`confirmed`, `in_progress`, `completed`, `cancelled_by_customer`, `cancelled_by_cleaner`, `disputed`) |
| limit | number | No | Results per page (default: 20) |
| cursor | string | No | Pagination cursor |

**Response (200):**
```json
{
  "bookings": [
    {
      "id": "uuid",
      "status": "confirmed",
      "cleaner": {
        "id": "uuid",
        "firstName": "Jane",
        "profileImageUrl": "https://..."
      },
      "startTime": "2024-01-20T09:00:00Z",
      "endTime": "2024-01-20T12:00:00Z",
      "total": 18360,
      "services": ["standard", "inside_oven"]
    }
  ],
  "meta": {
    "total": 5,
    "limit": 20
  }
}
```

---

### Payments

#### POST /payments/create-intent
**Description:** Create Stripe PaymentIntent for booking hold

**Request:**
```json
{
  "holdId": "uuid"
}
```

**Response (200):**
```json
{
  "clientSecret": "pi_xxx_secret_yyy",
  "paymentIntentId": "pi_xxx",
  "amount": 18360,
  "currency": "usd"
}
```

---

#### POST /stripe/webhook
**Description:** Stripe webhook endpoint (server-to-server)

**Headers:**
```
Stripe-Signature: <signature>
```

**Request:**
```json
{
  "id": "evt_xxx",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_xxx",
      "amount": 18360,
      "status": "succeeded"
    }
  }
}
```

**Response (200):**
```json
{
  "received": true
}
```

---

### Reviews

#### POST /reviews
**Description:** Submit a review for a completed booking

**Request:**
```json
{
  "bookingId": "uuid",
  "rating": 5,
  "text": "Jane did an amazing job! Very thorough and professional."
}
```

**Response (201):**
```json
{
  "review": {
    "id": "uuid",
    "bookingId": "uuid",
    "cleanerId": "uuid",
    "rating": 5,
    "text": "Jane did an amazing job!...",
    "createdAt": "2024-01-21T15:00:00Z"
  }
}
```

**Errors:**
- 400 Bad Request - Booking not completed
- 409 Conflict - Review already exists

---

#### POST /reviews/:id/report
**Description:** Report a review for moderation

**Request:**
```json
{
  "reason": "Inappropriate content"
}
```

**Response (201):**
```json
{
  "report": {
    "id": "uuid",
    "reviewId": "uuid",
    "status": "pending"
  }
}
```

---

### Cleaner Dashboard (Cleaner Role)

#### GET /cleaner/dashboard
**Description:** Get cleaner dashboard stats

**Response (200):**
```json
{
  "stats": {
    "monthlyEarnings": 4500,
    "upcomingBookings": 8,
    "rating": 4.8,
    "reviewCount": 42,
    "totalJobs": 120,
    "completionRate": 98.5
  },
  "earnings": {
    "thisMonth": 4500,
    "lastMonth": 5200,
    "pendingPayout": 650
  },
  "upcomingBookings": [
    {
      "id": "uuid",
      "customer": {
        "firstName": "Alice",
        "phone": "+14155559999"
      },
      "startTime": "2024-01-20T09:00:00Z",
      "endTime": "2024-01-20T12:00:00Z",
      "total": 16500,
      "services": ["standard"]
    }
  ]
}
```

---

#### POST /cleaner/profile
**Description:** Create/update cleaner profile

**Request:**
```json
{
  "bio": "Professional cleaner with 5 years experience...",
  "baseRate": 5500,
  "minHours": 2,
  "radiusMeters": 8000,
  "location": {
    "lat": 37.7749,
    "lng": -122.4194,
    "address": "San Francisco, CA"
  },
  "services": [
    { "type": "standard", "priceAddon": 0 },
    { "type": "deep_clean", "priceAddon": 1500 },
    { "type": "inside_oven", "priceAddon": 500 }
  ]
}
```

**Response (200):**
```json
{
  "cleaner": {
    "id": "uuid",
    "bio": "Professional cleaner...",
    "baseRate": 5500,
    "status": "active",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

---

#### POST /cleaner/availability
**Description:** Update weekly availability

**Request:**
```json
{
  "timezone": "America/Los_Angeles",
  "weeklySchedule": {
    "monday": [
      { "start": "09:00", "end": "17:00" }
    ],
    "tuesday": [
      { "start": "09:00", "end": "17:00" }
    ],
    "wednesday": [
      { "start": "09:00", "end": "12:00" },
      { "start": "14:00", "end": "18:00" }
    ],
    "thursday": [],
    "friday": [
      { "start": "09:00", "end": "17:00" }
    ],
    "saturday": [
      { "start": "10:00", "end": "16:00" }
    ],
    "sunday": []
  }
}
```

**Response (200):**
```json
{
  "availability": {
    "timezone": "America/Los_Angeles",
    "weeklySchedule": { /* ... */ },
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

---

#### POST /cleaner/availability/blackouts
**Description:** Add unavailable period (vacation, etc.)

**Request:**
```json
{
  "startTime": "2024-02-10T00:00:00Z",
  "endTime": "2024-02-17T23:59:59Z",
  "reason": "Vacation"
}
```

**Response (201):**
```json
{
  "blackout": {
    "id": "uuid",
    "startTime": "2024-02-10T00:00:00Z",
    "endTime": "2024-02-17T23:59:59Z",
    "reason": "Vacation"
  }
}
```

---

#### GET /cleaner/earnings
**Description:** Get earnings history

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| startDate | ISO 8601 | No | Filter start date |
| endDate | ISO 8601 | No | Filter end date |

**Response (200):**
```json
{
  "summary": {
    "totalEarnings": 24500,
    "totalBookings": 52,
    "averageBookingValue": 47115,
    "pendingPayout": 650
  },
  "earnings": [
    {
      "bookingId": "uuid",
      "customer": { "firstName": "Alice" },
      "completedAt": "2024-01-10T12:00:00Z",
      "amount": 16500,
      "payoutStatus": "succeeded",
      "paidAt": "2024-01-11T02:00:00Z"
    }
  ]
}
```

---

#### POST /cleaner/stripe-connect
**Description:** Create Stripe Connect onboarding link

**Response (200):**
```json
{
  "url": "https://connect.stripe.com/express/onboarding/xxx",
  "expiresAt": "2024-01-15T11:00:00Z"
}
```

---

### Admin (Admin Role)

#### GET /admin/stats
**Description:** Platform-wide statistics

**Response (200):**
```json
{
  "stats": {
    "totalUsers": 1250,
    "totalCleaners": 85,
    "activeCleaners": 72,
    "totalBookings": 3420,
    "gmv": 185000,
    "platformRevenue": 14800,
    "averageBookingValue": 5409
  },
  "growth": {
    "newUsersThisMonth": 42,
    "newCleanersThisMonth": 5,
    "bookingsThisMonth": 310
  }
}
```

---

#### GET /admin/disputes
**Description:** Get all disputes

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| status | string | No | Filter by status |
| limit | number | No | Results per page (default: 20) |

**Response (200):**
```json
{
  "disputes": [
    {
      "id": "uuid",
      "booking": {
        "id": "uuid",
        "startTime": "2024-01-20T09:00:00Z"
      },
      "customer": {
        "id": "uuid",
        "firstName": "John"
      },
      "cleaner": {
        "id": "uuid",
        "firstName": "Jane"
      },
      "reason": "Service quality issue",
      "status": "open",
      "createdAt": "2024-01-20T15:00:00Z"
    }
  ]
}
```

---

#### POST /admin/disputes/:id/resolve
**Description:** Resolve a dispute

**Request:**
```json
{
  "resolution": "resolved_refund",
  "refundAmount": 18360,
  "adminNotes": "Full refund issued due to..."
}
```

**Response (200):**
```json
{
  "dispute": {
    "id": "uuid",
    "status": "resolved_refund",
    "refundAmount": 18360,
    "resolvedAt": "2024-01-21T10:00:00Z"
  }
}
```

---

## Pagination

### Cursor-Based Pagination
All list endpoints support cursor-based pagination for consistent results.

**Request:**
```
GET /search/cleaners?limit=20&cursor=eyJpZCI6InV1aWQiLCJkaXN0YW5jZSI6MTI1MH0=
```

**Response:**
```json
{
  "data": [ /* results */ ],
  "meta": {
    "total": 100,
    "limit": 20,
    "cursor": "next_cursor_value"
  }
}
```

---

## Webhooks (Platform → Client)

### Webhook Events
Clients can subscribe to webhook events for real-time updates.

**Event Types:**
- `booking.confirmed`
- `booking.completed`
- `booking.cancelled`
- `payout.succeeded`
- `payout.failed`
- `review.created`

**Webhook Payload:**
```json
{
  "id": "evt_xxx",
  "type": "booking.confirmed",
  "createdAt": "2024-01-15T10:30:00Z",
  "data": {
    "bookingId": "uuid",
    "cleanerId": "uuid",
    "customerId": "uuid",
    "startTime": "2024-01-20T09:00:00Z"
  }
}
```

---

## Rate Limiting

**Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1234567890
```

**Rate Limit Response (429):**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests, please try again later",
    "retryAfter": 60
  }
}
```

---

## Idempotency

Idempotency is required for payment and booking mutations, and for auth sync upserts, via `X-Idempotency-Key`.

**Endpoints requiring `X-Idempotency-Key`:**
- `POST /auth/sync`
- `POST /booking/holds`
- `POST /bookings`
- `POST /bookings/:id/cancel`
- `POST /bookings/:id/complete`
- `POST /bookings/:id/dispute`
- Payment mutation endpoints (for example `POST /payments/create-intent`)

**Storage model (MVP):**
- Booking hold keys are persisted on `booking_holds.idempotency_key`.
- Stripe webhook deduplication is persisted on `webhook_events.id` (Stripe event ID).
- Other mutating endpoint keys are persisted in `api_idempotency_keys` (route + method + key unique tuple).

**Header format:**
```
X-Idempotency-Key: <uuid>
```

If the same idempotency key is used twice:
- First request: Processes normally
- Second request: Returns cached result from first request (same status code and body)

Keys expire after 24 hours.

---

## Versioning

API versions are specified in the URL path:
- Current: `/v1`
- Future: `/v2` (when breaking changes needed)

**Deprecation Policy:**
- 6 months notice before deprecating a version
- Sunset header on deprecated endpoints:
  ```
  Sunset: Sat, 31 Dec 2024 23:59:59 GMT
  ```

---

## Testing

### Sandbox Environment
**Base URL:** `https://api-sandbox.yourdomain.com/v1`

Test credentials:
- Email: `test@example.com`
- Password: `Test123!`

### Stripe Test Mode
Use Stripe test cards:
- Success: `4242 4242 4242 4242`
- Declined: `4000 0000 0000 0002`

---

## SDKs

### JavaScript/TypeScript
```typescript
import { CleaningAPI } from '@yourdomain/api-client'

const api = new CleaningAPI({
  apiKey: process.env.API_KEY,
  environment: 'production'
})

const cleaners = await api.cleaners.search({
  lat: 37.7749,
  lng: -122.4194,
  radiusMeters: 5000
})
```

### Python
```python
from cleaning_api import CleaningAPI

api = CleaningAPI(api_key=os.environ['API_KEY'])

cleaners = api.cleaners.search(
    lat=37.7749,
    lng=-122.4194,
    radius_meters=5000
)
```

---

## Postman Collection

Import the Postman collection for easy testing:
```
https://api.yourdomain.com/postman/collection.json
```

---

## Next Steps

See related docs:
- [Backend Architecture](./BACKEND_ARCHITECTURE.md)
- [Frontend Architecture](./FRONTEND_ARCHITECTURE.md)
- [Database Schema](./DATABASE_SCHEMA.md)
- [Infrastructure & Deployment](./INFRASTRUCTURE_DEPLOYMENT.md)
