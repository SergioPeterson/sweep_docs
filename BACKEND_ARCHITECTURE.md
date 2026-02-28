# Backend Architecture

## Stack Overview

### Core Technologies
- **Runtime:** Bun 1.x+
- **Language:** TypeScript 5.3+
- **Framework:** Elysia (modular monolith)
- **API Style:** REST (with GraphQL optional for future)
- **Validation:** Zod
- **ORM:** Prisma

### Alternative Stack (Higher Performance)
If Bun throughput becomes a bottleneck:
- **Runtime:** Go 1.21+
- **Framework:** Fiber or Gin
- **ORM:** GORM or sqlc

**Recommendation:** Start with Bun/Elysia for fast iteration and excellent performance. Bun's native TypeScript support and speed make it ideal for MVP and beyond.

---

## Architecture Pattern: Modular Monolith

### Structure
```
backend/
├── src/
│   ├── modules/
│   │   ├── auth/              # Authentication & authorization
│   │   ├── users/             # User management
│   │   ├── cleaners/          # Cleaner profiles & services
│   │   ├── availability/      # Availability rules & calendar
│   │   ├── search/            # Cleaner search & discovery
│   │   ├── booking/           # Booking lifecycle
│   │   ├── payments/          # Payment processing & webhooks
│   │   ├── payouts/           # Cleaner payouts
│   │   ├── reviews/           # Reviews & ratings
│   │   ├── messaging/         # In-app messaging (optional)
│   │   ├── bonuses/           # Bonus calculation & distribution
│   │   ├── notifications/     # Email, SMS, push notifications
│   │   ├── admin/             # Admin panel & support tools
│   │   └── analytics/         # Metrics & reporting
│   ├── common/
│   │   ├── decorators/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── pipes/
│   │   └── filters/
│   ├── infrastructure/
│   │   ├── database/
│   │   ├── cache/
│   │   ├── queue/
│   │   ├── storage/
│   │   └── third-party/
│   ├── config/
│   └── main.ts
├── workers/
│   ├── notification-worker.ts
│   ├── payout-worker.ts
│   ├── bonus-worker.ts
│   └── search-indexer.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── scripts/
```

### Module Boundaries

Each module follows Elysia conventions:
```
module-name/
├── index.ts                   # Module plugin definition
├── module-name.routes.ts      # HTTP endpoints
├── module-name.service.ts     # Business logic
├── module-name.repository.ts  # Data access
├── entities/                  # Domain models
│   └── module-name.entity.ts
├── dto/                       # Data transfer objects (Zod schemas)
│   ├── create-*.dto.ts
│   ├── update-*.dto.ts
│   └── query-*.dto.ts
└── tests/
    └── module-name.service.spec.ts
```

---

## Core Modules Deep Dive

### 1. Auth & Identity Module

**Responsibilities:**
- Validate Clerk-issued JWTs
- Map external auth identity to platform users
- Role-based access control
- Session-aware authorization checks

**Key Components:**
```typescript
// auth.service.ts
class AuthService {
  async getOrCreateUserFromToken(token: ClerkTokenPayload): Promise<AuthUser>
  async syncUser(dto: SyncUserDto): Promise<AuthUser>
  async requireRole(userId: string, allowedRoles: string[]): Promise<void>
}

// auth.plugin.ts (Elysia middleware)
import { Elysia } from 'elysia'
import { createRemoteJWKSet, jwtVerify } from 'jose'

export const authPlugin = new Elysia()
  .derive(async ({ headers, set }) => {
    const token = headers.authorization?.replace('Bearer ', '')
    if (!token) return { user: null }

    const JWKS = createRemoteJWKSet(new URL(process.env.CLERK_JWKS_URL!))
    const { payload } = await jwtVerify(token, JWKS)
    const user = await authService.getOrCreateUserFromToken(payload)

    return { user }
  })

// roles.guard.ts (Elysia guard pattern)
export const requireRole = (roles: string[]) =>
  (app: Elysia) => app.onBeforeHandle(({ user, set }) => {
    if (!user || !roles.includes(user.role)) {
      set.status = 403
      return { error: 'Forbidden' }
    }
  })
```

**Auth Flow:**
1. User authenticates with Clerk on web/mobile
2. Frontend sends Clerk bearer token to platform API
3. API verifies token signature via Clerk JWKS
4. API upserts/loads platform `users` row (`external_auth_id`)
5. Authorization checks run against platform roles

**Technology:**
- Token verification: `jose` + Clerk JWKS
- Identity provider: Clerk
- Role authorization: Platform DB (`users.role`)

---

### 2. Cleaner Profile Module

**Responsibilities:**
- Cleaner profile CRUD
- Service offerings and pricing
- Travel radius and coverage area
- Verification badges (background check, insurance)
- Profile image uploads

**Key Components:**
```typescript
class CleanerService {
  async createProfile(userId: string, dto: CreateCleanerDto): Promise<Cleaner>
  async updateProfile(cleanerId: string, dto: UpdateCleanerDto): Promise<Cleaner>
  async getProfile(cleanerId: string): Promise<CleanerProfile>
  async setServices(cleanerId: string, services: ServiceDto[]): Promise<void>
  async updateLocation(cleanerId: string, location: LocationDto): Promise<void>
  async uploadProfileImage(cleanerId: string, file: Buffer): Promise<string>
}

// DTOs
interface CreateCleanerDto {
  bio: string
  baseRate: number        // in cents
  minHours: number
  radiusMeters: number
  location: {
    lat: number
    lng: number
  }
}

interface ServiceDto {
  type: 'standard' | 'deep_clean' | 'move_in_out' | 'inside_fridge' | 'inside_oven'
  priceAddon: number     // in cents, can be 0
}
```

**Profile Image Upload Flow:**
1. Client uploads image (multipart/form-data)
2. Validate file type and size (max 5MB, JPEG/PNG)
3. Resize/optimize image (sharp library)
4. Upload to S3 with UUID key
5. Store S3 URL in database
6. Return signed CloudFront URL

---

### 3. Availability Module

**Responsibilities:**
- Weekly availability rules
- Blackout dates
- Real-time availability checking
- Next available slot computation

**Data Model:**
```typescript
interface AvailabilityRule {
  cleanerId: string
  timezone: string
  weeklyRules: {
    monday: TimeSlot[]
    tuesday: TimeSlot[]
    // ... etc
  }
  blackoutRanges: DateRange[]  // vacations, holidays
}

interface TimeSlot {
  start: string  // "09:00"
  end: string    // "17:00"
}

interface DateRange {
  start: Date
  end: Date
}
```

**Availability Check Algorithm:**
```typescript
class AvailabilityService {
  async isAvailable(
    cleanerId: string,
    requestedStart: Date,
    requestedEnd: Date
  ): Promise<boolean> {
    // 1. Load cleaner's availability rules
    const rules = await this.getAvailabilityRules(cleanerId)

    // 2. Check if requested time matches weekly rules
    const dayOfWeek = requestedStart.getDay()
    const matchesWeeklyRule = this.matchesWeeklyRule(
      rules.weeklyRules[dayOfWeek],
      requestedStart,
      requestedEnd
    )

    if (!matchesWeeklyRule) return false

    // 3. Check blackout dates
    const isBlackedOut = this.isInBlackoutRange(
      requestedStart,
      rules.blackoutRanges
    )

    if (isBlackedOut) return false

    // 4. Check for conflicting bookings (DB query)
    const hasConflict = await this.hasConflictingBooking(
      cleanerId,
      requestedStart,
      requestedEnd
    )

    return !hasConflict
  }
}
```

**Performance Optimization:**
- Cache availability rules per cleaner (Redis, 1 hour TTL)
- Precompute next 7 days of available slots (updated on booking/cancellation)
- Use database exclusion constraints for conflict detection

---

### 4. Search & Discovery Module

**Responsibilities:**
- Geo-based cleaner search
- Filtering (price, rating, services, availability)
- Ranking and sorting
- Search result caching

**Search Query:**
```typescript
interface SearchCleanersDto {
  lat: number
  lng: number
  radiusMeters?: number     // default 5000 (5km)
  date?: Date               // optional: filter by availability
  minRating?: number        // e.g., 4.5
  maxRate?: number          // in cents
  services?: string[]       // ['deep_clean', 'inside_oven']
  sortBy?: 'distance' | 'rating' | 'price'
  limit?: number            // default 20
  cursor?: string           // pagination
}

interface CleanerSearchResult {
  id: string
  name: string
  baseRate: number
  rating: number
  reviewCount: number
  distance: number          // meters
  profileImageUrl: string
  services: string[]
  nextAvailable?: Date
}
```

**Search Implementation (Phase 1 - Postgres):**
```sql
-- Using PostGIS for geo queries
SELECT
  c.id,
  c.name,
  c.base_rate,
  c.rating,
  c.review_count,
  ST_Distance(c.location, ST_MakePoint($lat, $lng)::geography) as distance,
  c.profile_image_url
FROM cleaners c
WHERE
  ST_DWithin(c.location, ST_MakePoint($lat, $lng)::geography, $radiusMeters)
  AND c.rating >= $minRating
  AND c.base_rate <= $maxRate
  AND c.status = 'active'
ORDER BY distance ASC
LIMIT $limit;
```

**Search Implementation (Phase 2 - OpenSearch):**
```typescript
class SearchService {
  async searchCleaners(query: SearchCleanersDto): Promise<CleanerSearchResult[]> {
    const searchQuery = {
      bool: {
        must: [
          { term: { status: 'active' } }
        ],
        filter: [
          {
            geo_distance: {
              distance: `${query.radiusMeters}m`,
              location: { lat: query.lat, lon: query.lng }
            }
          },
          { range: { rating: { gte: query.minRating || 0 } } },
          { range: { base_rate: { lte: query.maxRate || 999999 } } }
        ]
      }
    }

    const result = await this.opensearch.search({
      index: 'cleaners',
      body: { query: searchQuery, size: query.limit, sort: ['_geo_distance'] }
    })

    // Hydrate full details from Postgres (cache in Redis)
    return this.hydrateCleaners(result.hits.hits.map(h => h._id))
  }
}
```

**Search Ranking Algorithm:**

Default sort (`sortBy=relevance`) uses a weighted composite score:

```typescript
/**
 * Ranking formula:
 *   score = (distanceScore * 0.30)
 *         + (ratingScore   * 0.35)
 *         + (availScore    * 0.20)
 *         + (volumeScore   * 0.15)
 *
 * Each sub-score is normalized to 0-100.
 */
function calculateRelevanceScore(cleaner: CleanerSearchRow, query: SearchCleanersDto): number {
  // Distance: closer = higher. Linear decay to 0 at max radius.
  const distanceScore = Math.max(0, 100 * (1 - cleaner.distance / query.radiusMeters))

  // Rating: 5.0 → 100, 4.0 → 80, min 4.0 to appear
  const ratingScore = Math.min(100, cleaner.rating * 20)

  // Availability: available on requested date = 100, otherwise 50
  const availScore = cleaner.availableOnDate ? 100 : 50

  // Volume: log scale based on completed bookings (caps at 200 bookings)
  const volumeScore = Math.min(100, (Math.log2(cleaner.totalBookings + 1) / Math.log2(201)) * 100)

  return (distanceScore * 0.30)
       + (ratingScore   * 0.35)
       + (availScore    * 0.20)
       + (volumeScore   * 0.15)
}
```

**Weight rationale:**
- **Rating (35%)** - Strongest signal of quality, drives repeat bookings
- **Distance (30%)** - Proximity matters for on-time arrival and travel costs
- **Availability (20%)** - Cleaners available on the requested date rank higher
- **Volume (15%)** - Track record proxy; log scale prevents incumbents from dominating

**New cleaner boost:** Cleaners with <5 bookings get a temporary +10 bonus to their score for their first 30 days to aid discovery.

**Caching Strategy:**
- Cache popular search results (lat/lng grid + common filters) for 10 minutes
- Cache individual cleaner profiles for 1 hour
- Invalidate cache on profile update

---

### 5. Booking Module

**Responsibilities:**
- Booking holds (temporary reservation)
- Booking confirmation (after payment)
- Cancellation and refunds
- Booking status lifecycle
- Double-booking prevention

**Booking Lifecycle:**
```
REQUESTED → HOLD_CREATED → PAYMENT_PENDING → CONFIRMED
                ↓
            EXPIRED (if not paid within 5 min)

CONFIRMED → IN_PROGRESS → COMPLETED → REVIEWED

CONFIRMED → CANCELLED_BY_CUSTOMER
CONFIRMED → CANCELLED_BY_CLEANER
```

**Critical: Double-Booking Prevention**

**Database Constraint (Postgres):**
```sql
-- Exclusion constraint prevents overlapping bookings
CREATE TABLE booking (
  id UUID PRIMARY KEY,
  cleaner_id UUID NOT NULL,
  start_time TIMESTAMPTZ NOT NULL,
  end_time TIMESTAMPTZ NOT NULL,
  status VARCHAR NOT NULL,
  -- other fields...

  CONSTRAINT no_overlap_confirmed
    EXCLUDE USING GIST (
      cleaner_id WITH =,
      tstzrange(start_time, end_time) WITH &&
    )
    WHERE (status IN ('confirmed', 'in_progress', 'completed'))
);

-- Separate table for holds (short-lived)
CREATE TABLE booking_hold (
  id UUID PRIMARY KEY,
  cleaner_id UUID NOT NULL,
  start_time TIMESTAMPTZ NOT NULL,
  end_time TIMESTAMPTZ NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  idempotency_key VARCHAR UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Exclusion constraint for holds
ALTER TABLE booking_hold
  ADD CONSTRAINT no_overlap_holds
  EXCLUDE USING GIST (
    cleaner_id WITH =,
    tstzrange(start_time, end_time) WITH &&
  );

-- Cleanup worker removes expired holds every minute
DELETE FROM booking_hold WHERE expires_at < NOW();
```

**Booking Service:**
```typescript
class BookingService {
  async createHold(dto: CreateBookingHoldDto): Promise<BookingHold> {
    // Idempotency: check if hold already exists
    const existing = await this.holdRepository.findByIdempotencyKey(dto.idempotencyKey)
    if (existing) return existing

    // Create hold with 5-minute expiry
    const hold = await this.holdRepository.create({
      ...dto,
      expiresAt: new Date(Date.now() + 5 * 60 * 1000)
    })

    // Schedule cleanup job
    await this.queueService.scheduleHoldExpiry(hold.id, hold.expiresAt)

    return hold
  }

  async confirmBooking(
    holdId: string,
    paymentIntentId: string
  ): Promise<Booking> {
    return this.db.transaction(async (tx) => {
      // 1. Verify hold exists and not expired
      const hold = await tx.bookingHold.findUnique({ where: { id: holdId } })
      if (!hold || hold.expiresAt < new Date()) {
        throw new BadRequestException('Hold expired')
      }

      // 2. Verify payment succeeded
      const payment = await this.stripeService.getPaymentIntent(paymentIntentId)
      if (payment.status !== 'succeeded') {
        throw new BadRequestException('Payment not confirmed')
      }

      // 3. Create confirmed booking
      const booking = await tx.booking.create({
        data: {
          cleanerId: hold.cleanerId,
          customerId: hold.customerId,
          startTime: hold.startTime,
          endTime: hold.endTime,
          status: 'confirmed',
          priceTotal: payment.amount,
          platformFee: this.calculatePlatformFee(payment.amount),
          paymentIntentId
        }
      })

      // 4. Delete hold
      await tx.bookingHold.delete({ where: { id: holdId } })

      // 5. Emit events (async)
      this.eventEmitter.emit('booking.confirmed', booking)

      return booking
    })
  }

  async cancelBooking(bookingId: string, reason: string): Promise<void> {
    const booking = await this.bookingRepository.findById(bookingId)

    // Check cancellation policy
    const hoursUntilStart = (booking.startTime.getTime() - Date.now()) / (1000 * 60 * 60)
    const refundAmount = this.calculateRefund(booking.priceTotal, hoursUntilStart)

    await this.db.transaction(async (tx) => {
      // Update booking status
      await tx.booking.update({
        where: { id: bookingId },
        data: { status: 'cancelled_by_customer', cancellationReason: reason }
      })

      // Process refund if applicable
      if (refundAmount > 0) {
        await this.paymentsService.refund(booking.paymentIntentId, refundAmount)
      }

      // Notify parties
      this.eventEmitter.emit('booking.cancelled', { booking, refundAmount })
    })
  }

  private calculateRefund(totalAmount: number, hoursUntilStart: number): number {
    // Example policy:
    // >24 hours: 100% refund
    // 12-24 hours: 50% refund
    // <12 hours: 0% refund
    if (hoursUntilStart >= 24) return totalAmount
    if (hoursUntilStart >= 12) return Math.floor(totalAmount * 0.5)
    return 0
  }
}
```

**Idempotency:**
- All booking mutations accept an `idempotency_key`
- Store key in database with unique constraint
- Return existing result if duplicate request detected

---

### 6. Payments Module

**Responsibilities:**
- Stripe PaymentIntent creation
- Webhook processing
- Refund handling
- Payment reconciliation

**Payment Flow:**
```typescript
class PaymentsService {
  async createPaymentIntent(dto: CreatePaymentDto): Promise<PaymentIntent> {
    const { bookingHoldId, amount, customerId } = dto

    // Create Stripe PaymentIntent
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount,
      currency: 'usd',
      customer: customerId,
      metadata: { bookingHoldId },
      automatic_payment_methods: { enabled: true }
    })

    // Store in database
    await this.paymentRepository.create({
      id: paymentIntent.id,
      bookingHoldId,
      amount,
      status: 'pending',
      stripePaymentIntentId: paymentIntent.id
    })

    return paymentIntent
  }

  async handleWebhook(event: Stripe.Event): Promise<void> {
    // Idempotency: check if event already processed
    const existing = await this.webhookEventRepository.findById(event.id)
    if (existing) {
      console.log(`Event ${event.id} already processed`)
      return
    }

    // Store event
    await this.webhookEventRepository.create({
      id: event.id,
      type: event.type,
      data: event.data,
      processedAt: new Date()
    })

    // Process event
    switch (event.type) {
      case 'payment_intent.succeeded':
        await this.handlePaymentSucceeded(event.data.object)
        break
      case 'payment_intent.payment_failed':
        await this.handlePaymentFailed(event.data.object)
        break
      // ... other events
    }
  }

  private async handlePaymentSucceeded(paymentIntent: Stripe.PaymentIntent): Promise<void> {
    const bookingHoldId = paymentIntent.metadata.bookingHoldId

    // Trigger booking confirmation
    await this.bookingService.confirmBooking(bookingHoldId, paymentIntent.id)
  }
}
```

**Webhook Security:**
- Verify Stripe signature
- Store event ID to prevent replay
- Process via queue (SQS) for reliability
- Manual reconciliation job (daily) for safety

**Payment-to-Booking Failure Recovery:**

If Stripe charges succeed but the booking confirmation DB write fails, the system must recover gracefully:

```typescript
class PaymentReconciliationService {
  /**
   * Runs every 5 minutes via scheduled task.
   * Finds payments that succeeded in Stripe but have no confirmed booking.
   */
  async reconcileOrphanedPayments(): Promise<void> {
    // 1. Find payments marked 'succeeded' with no matching confirmed booking
    const orphaned = await this.paymentRepository.findOrphaned({
      status: 'succeeded',
      olderThan: new Date(Date.now() - 5 * 60 * 1000), // at least 5 min old
      hasNoConfirmedBooking: true
    })

    for (const payment of orphaned) {
      try {
        // 2. Check if hold still exists and is valid
        const hold = await this.holdRepository.findByPaymentIntent(
          payment.stripePaymentIntentId
        )

        if (hold) {
          // 3a. Hold exists - retry booking confirmation
          await this.bookingService.confirmBooking(hold.id, payment.stripePaymentIntentId)
          logger.info('Reconciled orphaned payment', { paymentId: payment.id })
        } else {
          // 3b. Hold expired - issue automatic refund
          await this.stripe.refunds.create({
            payment_intent: payment.stripePaymentIntentId,
            reason: 'requested_by_customer'
          })
          await this.paymentRepository.update(payment.id, {
            status: 'refunded',
            failureReason: 'hold_expired_during_confirmation'
          })
          logger.warn('Refunded orphaned payment - hold expired', { paymentId: payment.id })
        }
      } catch (error) {
        // 4. Alert on repeated failures
        Sentry.captureException(error, { extra: { paymentId: payment.id } })
        await this.alertService.notify('orphaned-payment-recovery-failed', {
          paymentId: payment.id,
          amount: payment.amount
        })
      }
    }
  }
}
```

**Recovery guarantees:**
- No customer is ever charged without a booking or refund
- Reconciliation runs on a scheduled cadence (every 5 min)
- Failed recoveries trigger alerts for manual intervention
- All recovery actions are audit-logged

---

### 7. Payouts Module

**Responsibilities:**
- Cleaner payout calculation
- Stripe Connect transfers
- Payout retry logic
- Payout history

**Payout Flow:**
```typescript
class PayoutsService {
  async processPayoutForBooking(bookingId: string): Promise<Payout> {
    const booking = await this.bookingRepository.findById(bookingId)

    if (booking.status !== 'completed') {
      throw new BadRequestException('Booking not completed')
    }

    // Calculate payout amount (cleaner rate × hours)
    const hours = (booking.endTime.getTime() - booking.startTime.getTime()) / (1000 * 60 * 60)
    const cleaner = await this.cleanerRepository.findById(booking.cleanerId)
    const payoutAmount = cleaner.baseRate * hours

    // Create Stripe transfer to cleaner's Connect account
    const transfer = await this.stripe.transfers.create({
      amount: payoutAmount,
      currency: 'usd',
      destination: cleaner.stripeConnectAccountId,
      metadata: { bookingId }
    })

    // Store payout record
    const payout = await this.payoutRepository.create({
      bookingId,
      cleanerId: booking.cleanerId,
      amount: payoutAmount,
      stripeTransferId: transfer.id,
      status: 'processing'
    })

    return payout
  }

  async retryFailedPayouts(): Promise<void> {
    const failedPayouts = await this.payoutRepository.findFailed()

    for (const payout of failedPayouts) {
      try {
        await this.processPayoutForBooking(payout.bookingId)
      } catch (error) {
        console.error(`Payout retry failed for ${payout.id}:`, error)
      }
    }
  }
}
```

**Payout Triggers:**
- Automatic: 24 hours after booking completion (customer auto-confirm)
- Manual: Customer confirms completion immediately
- Batch: Daily job processes all pending payouts

---

### 8. Reviews Module

**Responsibilities:**
- Submit reviews (customers → cleaners)
- Review integrity checks
- Rating aggregation
- Review moderation

**Review Service:**
```typescript
class ReviewsService {
  async submitReview(dto: CreateReviewDto): Promise<Review> {
    const { bookingId, rating, text } = dto

    // Validate booking exists and customer is authorized
    const booking = await this.bookingRepository.findById(bookingId)
    if (booking.status !== 'completed') {
      throw new BadRequestException('Can only review completed bookings')
    }

    // Check if review already exists
    const existing = await this.reviewRepository.findByBookingId(bookingId)
    if (existing) {
      throw new ConflictException('Review already submitted')
    }

    // Create review
    const review = await this.reviewRepository.create({
      bookingId,
      cleanerId: booking.cleanerId,
      customerId: booking.customerId,
      rating,
      text
    })

    // Update cleaner's aggregate rating (async)
    await this.updateCleanerRating(booking.cleanerId)

    return review
  }

  private async updateCleanerRating(cleanerId: string): Promise<void> {
    const stats = await this.reviewRepository.getStats(cleanerId)

    await this.cleanerRepository.update(cleanerId, {
      rating: stats.averageRating,
      reviewCount: stats.totalReviews
    })
  }
}
```

---

### 9. Bonuses Module

**Responsibilities:**
- Calculate top cleaner scores
- Allocate bonus pool
- Distribute bonuses

**Bonus Calculation:**
```typescript
class BonusesService {
  async calculateMonthlyBonuses(year: number, month: number): Promise<void> {
    const periodStart = new Date(year, month - 1, 1)
    const periodEnd = new Date(year, month, 0, 23, 59, 59)

    // 1. Calculate total GMV for period
    const gmv = await this.bookingRepository.sumTotalPrice(periodStart, periodEnd)
    const bonusPool = gmv * 0.015  // 1.5% of GMV

    // 2. Calculate scores for all cleaners
    const cleaners = await this.cleanerRepository.findActive()
    const scores = await Promise.all(
      cleaners.map(c => this.calculateCleanerScore(c.id, periodStart, periodEnd))
    )

    // 3. Distribute bonus proportionally to top 20%
    const topCleaners = scores
      .sort((a, b) => b.score - a.score)
      .slice(0, Math.ceil(scores.length * 0.2))

    const totalTopScore = topCleaners.reduce((sum, c) => sum + c.score, 0)

    for (const cleaner of topCleaners) {
      const bonusAmount = Math.floor((cleaner.score / totalTopScore) * bonusPool)
      await this.distributeBonus(cleaner.cleanerId, bonusAmount, year, month)
    }
  }

  private async calculateCleanerScore(
    cleanerId: string,
    start: Date,
    end: Date
  ): Promise<{ cleanerId: string; score: number }> {
    const [rating, repeatRate, hours, cancellations] = await Promise.all([
      this.reviewRepository.getAverageRating(cleanerId, start, end),
      this.bookingRepository.getRepeatBookingRate(cleanerId, start, end),
      this.bookingRepository.getTotalHours(cleanerId, start, end),
      this.bookingRepository.getCancellationCount(cleanerId, start, end)
    ])

    const score =
      (rating * 50) +
      (repeatRate * 30) +
      (hours * 0.2) -
      (cancellations * 20)

    return { cleanerId, score }
  }
}
```

---

### 10. Notifications Module

**Responsibilities:**
- Send emails (transactional + marketing)
- Send SMS (booking confirmations, reminders)
- Send push notifications (mobile)
- Template management
- Delivery tracking

**Notification Service:**
```typescript
class NotificationsService {
  async sendBookingConfirmation(booking: Booking): Promise<void> {
    const [cleaner, customer] = await Promise.all([
      this.cleanerRepository.findById(booking.cleanerId),
      this.userRepository.findById(booking.customerId)
    ])

    // Send to customer
    await Promise.all([
      this.emailService.send({
        to: customer.email,
        template: 'booking-confirmed-customer',
        data: { booking, cleaner }
      }),
      this.smsService.send({
        to: customer.phone,
        message: `Booking confirmed with ${cleaner.name} on ${booking.startTime}`
      })
    ])

    // Send to cleaner
    await Promise.all([
      this.emailService.send({
        to: cleaner.email,
        template: 'booking-confirmed-cleaner',
        data: { booking, customer }
      }),
      this.smsService.send({
        to: cleaner.phone,
        message: `New booking from ${customer.name} on ${booking.startTime}`
      })
    ])
  }

  async sendReminder(bookingId: string): Promise<void> {
    const booking = await this.bookingRepository.findById(bookingId)
    const hoursUntil = (booking.startTime.getTime() - Date.now()) / (1000 * 60 * 60)

    // Send reminder 24 hours before
    if (hoursUntil <= 24 && hoursUntil > 23) {
      await this.sendBookingReminder(booking)
    }
  }
}
```

**Queue-Based Processing:**
- All notifications sent via SQS queue
- Retry with exponential backoff
- Dead-letter queue for failed notifications
- Rate limiting per user (prevent spam)

---

## Real-Time Updates (Server-Sent Events)

### Why SSE over WebSockets

- **Simpler** - unidirectional (server → client), no handshake complexity
- **HTTP-native** - works through ALB, CDN, proxies without special config
- **Auto-reconnect** - built into the browser `EventSource` API
- **Sufficient** - booking status updates don't need bidirectional communication

WebSockets can be added later if in-app chat is needed.

### SSE Endpoint

```typescript
// modules/events/events.routes.ts
import { Elysia } from 'elysia'

export const eventsRoutes = new Elysia({ prefix: '/events' })
  .get('/stream', async function* ({ user, set }) {
    if (!user) {
      set.status = 401
      return
    }

    set.headers['content-type'] = 'text/event-stream'
    set.headers['cache-control'] = 'no-cache'
    set.headers['connection'] = 'keep-alive'

    // Subscribe to user-specific Redis channel
    const channel = `user:${user.id}:events`
    const subscriber = redis.duplicate()
    await subscriber.subscribe(channel)

    try {
      // Heartbeat every 30s to keep connection alive
      const heartbeat = setInterval(() => {}, 30000)

      for await (const message of subscriber) {
        const event = JSON.parse(message)
        yield `event: ${event.type}\ndata: ${JSON.stringify(event.data)}\n\n`
      }

      clearInterval(heartbeat)
    } finally {
      await subscriber.unsubscribe(channel)
      await subscriber.quit()
    }
  })
```

### Event Types

| Event | Recipient | Trigger |
|-------|-----------|---------|
| `booking.confirmed` | Customer + Cleaner | Payment succeeds |
| `booking.cancelled` | Customer + Cleaner | Either party cancels |
| `booking.completed` | Customer | Cleaner marks complete |
| `booking.reminder` | Customer + Cleaner | 24hr before booking |
| `payout.succeeded` | Cleaner | Stripe transfer completes |
| `review.received` | Cleaner | Customer submits review |

### Publishing Events

```typescript
// common/events/event-publisher.ts
class EventPublisher {
  async publish(userId: string, event: { type: string; data: any }): Promise<void> {
    const channel = `user:${userId}:events`
    await this.redis.publish(channel, JSON.stringify(event))
  }

  async publishBookingUpdate(booking: Booking): Promise<void> {
    const event = {
      type: `booking.${booking.status}`,
      data: {
        bookingId: booking.id,
        status: booking.status,
        startTime: booking.startTime,
        updatedAt: new Date().toISOString()
      }
    }

    // Notify both parties
    await Promise.all([
      this.publish(booking.customerId, event),
      this.publish(booking.cleanerId, event)
    ])
  }
}
```

### Frontend Integration

```typescript
// lib/hooks/useEventStream.ts
export function useEventStream(onEvent: (event: { type: string; data: any }) => void) {
  useEffect(() => {
    const token = getAuthToken()
    const source = new EventSource(`${API_URL}/events/stream`, {
      headers: { Authorization: `Bearer ${token}` }
    })

    source.addEventListener('booking.confirmed', (e) => {
      onEvent({ type: 'booking.confirmed', data: JSON.parse(e.data) })
      queryClient.invalidateQueries({ queryKey: ['bookings'] })
    })

    source.addEventListener('payout.succeeded', (e) => {
      onEvent({ type: 'payout.succeeded', data: JSON.parse(e.data) })
      queryClient.invalidateQueries({ queryKey: ['earnings'] })
    })

    source.onerror = () => {
      // EventSource auto-reconnects; log for observability
      logger.warn('SSE connection lost, reconnecting...')
    }

    return () => source.close()
  }, [])
}
```

### Scaling Considerations

- **MVP:** Single API instance holds SSE connections directly via Redis Pub/Sub
- **Scale:** If >1000 concurrent SSE connections, extract to a dedicated lightweight SSE gateway service
- **Fallback:** Clients that don't support SSE fall back to polling `/me/bookings` every 30s

---

## Scheduled Jobs

### Job Scheduler Design

Scheduled jobs run as **ECS Scheduled Tasks** triggered by **EventBridge Scheduler**. Each job is a short-lived Fargate task that runs a specific command, then exits.

**Why EventBridge + ECS (not in-process cron):**
- Jobs survive API deployments and restarts
- Independent scaling from API service
- Built-in retry and failure alerting
- No risk of duplicate execution from multiple API instances

### Job Registry

| Job | Schedule | Description | Timeout |
|-----|----------|-------------|---------|
| `hold-cleanup` | Every 1 min | Delete expired booking holds | 30s |
| `booking-reminders` | Every 15 min | Send 24hr-before reminders | 2 min |
| `payment-reconciliation` | Every 5 min | Reconcile orphaned payments (see Payments Module) | 2 min |
| `auto-confirm-bookings` | Every 30 min | Auto-confirm completed bookings after 24hr | 2 min |
| `payout-processor` | Every 1 hour | Process pending payouts batch | 5 min |
| `bonus-calculator` | 1st of month, 3 AM | Calculate and distribute monthly bonuses | 10 min |
| `rating-refresh` | Daily, 2 AM | Refresh materialized views (top_cleaners) | 5 min |
| `stale-hold-alert` | Every 5 min | Alert if hold cleanup is falling behind | 30s |

### Job Runner

```typescript
// workers/job-runner.ts
const JOBS: Record<string, () => Promise<void>> = {
  'hold-cleanup': async () => {
    const deleted = await db.bookingHold.deleteMany({
      where: { expiresAt: { lt: new Date() } }
    })
    logger.info(`Cleaned up ${deleted.count} expired holds`)
  },

  'booking-reminders': async () => {
    const tomorrow = new Date(Date.now() + 24 * 60 * 60 * 1000)
    const upcoming = await db.booking.findMany({
      where: {
        startTime: { gte: new Date(), lte: tomorrow },
        status: 'confirmed',
        reminderSentAt: null
      }
    })
    for (const booking of upcoming) {
      await notificationService.sendReminder(booking.id)
      await db.booking.update({
        where: { id: booking.id },
        data: { reminderSentAt: new Date() }
      })
    }
  },

  'auto-confirm-bookings': async () => {
    const cutoff = new Date(Date.now() - 24 * 60 * 60 * 1000)
    const unconfirmed = await db.booking.findMany({
      where: {
        status: 'completed',
        customerConfirmed: false,
        completedAt: { lt: cutoff }
      }
    })
    for (const booking of unconfirmed) {
      await bookingService.autoConfirm(booking.id)
    }
  },

  'payment-reconciliation': async () => {
    await paymentReconciliationService.reconcileOrphanedPayments()
  }
}

// Entrypoint: bun run workers/job-runner.ts <job-name>
const jobName = process.argv[2]
if (!JOBS[jobName]) {
  console.error(`Unknown job: ${jobName}`)
  process.exit(1)
}

await JOBS[jobName]()
process.exit(0)
```

### Observability

- Each job emits a CloudWatch metric on start/success/failure
- Job duration tracked as a histogram
- Alert if any job fails 3 consecutive times
- Alert if `hold-cleanup` hasn't run in 5+ minutes

---

## Infrastructure Integration

### Database (Prisma)

**Connection Pool Configuration:**
```typescript
// database.config.ts
export const databaseConfig = {
  url: process.env.DATABASE_URL,
  connectionLimit: 20,          // for API service
  pool: {
    min: 2,
    max: 20,
    acquireTimeoutMillis: 30000,
    idleTimeoutMillis: 30000
  },
  log: ['error', 'warn'],
  errorFormat: 'minimal'
}
```

### Cache (Redis)

**Usage Patterns:**
```typescript
class CacheService {
  async get<T>(key: string): Promise<T | null> {
    const value = await this.redis.get(key)
    return value ? JSON.parse(value) : null
  }

  async set(key: string, value: any, ttlSeconds: number): Promise<void> {
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value))
  }

  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern)
    if (keys.length > 0) {
      await this.redis.del(...keys)
    }
  }
}

// Cache keys:
// - cleaner:{id}              → 1 hour TTL
// - cleaner:{id}:availability → 15 min TTL
// - search:{hash}             → 10 min TTL
// - user:{id}                 → 30 min TTL
```

### Queue (SQS)

**Queue Types:**
```typescript
enum QueueName {
  NOTIFICATIONS = 'notifications-queue',
  PAYOUTS = 'payouts-queue',
  SEARCH_INDEX = 'search-index-queue',
  WEBHOOKS = 'webhooks-queue'
}

class QueueService {
  async publish(queue: QueueName, message: any, delay?: number): Promise<void> {
    await this.sqs.sendMessage({
      QueueUrl: this.getQueueUrl(queue),
      MessageBody: JSON.stringify(message),
      DelaySeconds: delay || 0
    })
  }

  async consume(queue: QueueName, handler: (msg: any) => Promise<void>): Promise<void> {
    while (true) {
      const messages = await this.sqs.receiveMessage({
        QueueUrl: this.getQueueUrl(queue),
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20  // long polling
      })

      for (const msg of messages.Messages || []) {
        try {
          await handler(JSON.parse(msg.Body))
          await this.sqs.deleteMessage({
            QueueUrl: this.getQueueUrl(queue),
            ReceiptHandle: msg.ReceiptHandle
          })
        } catch (error) {
          console.error('Queue processing error:', error)
          // Message will become visible again after visibility timeout
        }
      }
    }
  }
}
```

---

## Error Handling

### Global Error Handler (Elysia)
```typescript
import { Elysia } from 'elysia'
import * as Sentry from '@sentry/bun'

export const errorHandler = new Elysia()
  .onError(({ code, error, set, path }) => {
    const status = code === 'VALIDATION' ? 400
      : code === 'NOT_FOUND' ? 404
      : code === 'PARSE' ? 400
      : 500

    const message = error.message || 'Internal server error'

    // Log to Sentry
    if (status >= 500) {
      Sentry.captureException(error)
    }

    set.status = status
    return {
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path
    }
  })
```

### Custom Exceptions
```typescript
export class BookingConflictError extends Error {
  constructor() {
    super('This time slot is no longer available')
    this.name = 'BookingConflictError'
  }
}

export class PaymentFailedError extends Error {
  constructor(reason: string) {
    super(`Payment failed: ${reason}`)
    this.name = 'PaymentFailedError'
  }
}
```

---

## Testing Strategy

### Unit Tests (Bun Test)
```typescript
import { describe, it, expect, mock, beforeEach } from 'bun:test'
import { BookingService } from './booking.service'

describe('BookingService', () => {
  let service: BookingService
  let mockRepository: any

  beforeEach(() => {
    mockRepository = {
      create: mock(() => Promise.resolve({ id: '456' })),
      findById: mock(() => Promise.resolve(null))
    }
    service = new BookingService(mockRepository)
  })

  it('should create booking hold', async () => {
    const dto = { cleanerId: '123', startTime: new Date(), endTime: new Date() }
    mockRepository.create.mockResolvedValue({ id: '456', ...dto })

    const result = await service.createHold(dto)
    expect(result.id).toBe('456')
  })
})
```

### Integration Tests (Elysia + Bun Test)
```typescript
import { describe, it, expect, beforeAll } from 'bun:test'
import { app } from '../src/app'

describe('Booking API (e2e)', () => {
  it('/POST booking/holds', async () => {
    const response = await app.handle(
      new Request('http://localhost/booking/holds', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          cleanerId: '123',
          startTime: '2024-01-01T10:00:00Z',
          endTime: '2024-01-01T13:00:00Z'
        })
      })
    )

    expect(response.status).toBe(201)
    const body = await response.json()
    expect(body.id).toBeDefined()
  })
})
```

---

## Performance Optimization

### N+1 Query Prevention
```typescript
// BAD: N+1 queries
async getBatch(ids: string[]): Promise<Cleaner[]> {
  const cleaners = []
  for (const id of ids) {
    const cleaner = await this.cleanerRepo.findById(id)
    cleaners.push(cleaner)
  }
  return cleaners
}

// GOOD: Single query
async getBatch(ids: string[]): Promise<Cleaner[]> {
  return this.cleanerRepo.findMany({ where: { id: { in: ids } } })
}
```

### DataLoader Pattern (for GraphQL if added)
```typescript
const cleanerLoader = new DataLoader(async (ids: string[]) => {
  const cleaners = await cleanerRepo.findMany({ where: { id: { in: ids } } })
  return ids.map(id => cleaners.find(c => c.id === id))
})
```

### Database Indexes
See DATABASE_SCHEMA.md for full index strategy

---

## Security Best Practices

1. **Input Validation:** Validate all DTOs with Zod or class-validator
2. **SQL Injection:** Use Prisma (ORM prevents SQL injection)
3. **XSS:** Sanitize user input before storing/displaying
4. **CSRF:** Not needed for API (stateless JWT auth)
5. **Rate Limiting:** Implement per-endpoint and per-user rate limits
6. **Secrets Management:** Use AWS Secrets Manager or environment variables (never commit)
7. **Audit Logging:** Log all state-changing operations

---

## Monitoring & Observability

### Metrics to Track
- Request rate (req/sec)
- Error rate (%)
- Latency (p50, p95, p99)
- Database connection pool usage
- Queue depth
- Cache hit rate

### Logging
```typescript
// Structured logging with pino or console
import { logger } from './lib/logger'

logger.info('Booking created', {
  bookingId: booking.id,
  cleanerId: booking.cleanerId,
  customerId: booking.customerId,
  amount: booking.priceTotal
})
```

### Alerts
- API error rate >1%
- p95 latency >1s
- Database CPU >80%
- Queue depth >1000
- Failed payouts

---

## Next Steps

See related docs:
- [Database Schema](./DATABASE_SCHEMA.md)
- [API Specification](./API_SPECIFICATION.md)
- [Infrastructure & Deployment](./INFRASTRUCTURE_DEPLOYMENT.md)
