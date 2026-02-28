# Database Schema

## Database Technology

**PostgreSQL 15+** with extensions:
- **PostGIS** - Geospatial queries (location-based search)
- **pg_trgm** - Trigram matching (fuzzy text search)
- **uuid-ossp** - UUID generation
- **btree_gist** - Exclusion constraints for time ranges

---

## Schema Overview

```
Core Tables:
├── users                    # All users (customers, cleaners, admins)
├── cleaners                 # Cleaner-specific data
├── cleaner_services         # Services offered by cleaners
├── availability_rules       # Weekly availability patterns
├── availability_blackouts   # Vacation/unavailable periods

Booking Tables:
├── booking_holds            # Temporary reservations (5 min TTL)
├── bookings                 # Confirmed bookings
├── booking_services         # Selected services per booking

Payment Tables:
├── payments                 # Payment records (Stripe)
├── payouts                  # Cleaner payouts (Stripe Connect)
├── refunds                  # Refund records

Review & Reputation:
├── reviews                  # Customer → Cleaner reviews
├── review_reports           # Flagged reviews

Bonus System:
├── bonus_periods            # Monthly/yearly bonus periods
├── bonus_awards             # Individual bonus distributions

Admin & Support:
├── audit_logs               # All state-changing actions
├── disputes                 # Customer disputes
├── support_tickets          # Support requests
├── webhook_events           # Stripe webhook deduplication
```

---

## Core Tables

### users

**Primary table for all users (polymorphic - customers, cleaners, admins).**

```sql
CREATE TYPE user_role AS ENUM ('customer', 'cleaner', 'admin');
CREATE TYPE user_status AS ENUM ('active', 'suspended', 'deleted');

CREATE TABLE users (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  external_auth_id  VARCHAR(128) UNIQUE,  -- Clerk user ID
  email             VARCHAR(255) NOT NULL UNIQUE,
  phone             VARCHAR(20) UNIQUE,
  phone_verified    BOOLEAN DEFAULT FALSE,
  email_verified    BOOLEAN DEFAULT FALSE,
  role              user_role NOT NULL DEFAULT 'customer',
  status            user_status NOT NULL DEFAULT 'active',
  first_name        VARCHAR(100),
  last_name         VARCHAR(100),
  profile_image_url TEXT,
  stripe_customer_id VARCHAR(100) UNIQUE,  -- Stripe Customer ID
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  updated_at        TIMESTAMPTZ DEFAULT NOW(),
  last_login_at     TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_stripe_customer_id ON users(stripe_customer_id);
CREATE INDEX idx_users_role_status ON users(role, status);
```

**Notes:**
- `role` is single-role in MVP (`customer`, `cleaner`, `admin`)
- If multi-role is needed later, migrate to a `user_roles` junction table
- `external_auth_id` maps platform users to Clerk identities
- `stripe_customer_id` links to Stripe's Customer object

---

### cleaners

**Cleaner-specific profile data (extends users where role='cleaner').**

```sql
CREATE TYPE cleaner_status AS ENUM ('pending', 'active', 'inactive', 'suspended');

CREATE TABLE cleaners (
  user_id                  UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
  bio                      TEXT,
  base_rate                INTEGER NOT NULL,  -- cents per hour
  min_hours                INTEGER NOT NULL DEFAULT 2,
  radius_meters            INTEGER NOT NULL DEFAULT 5000,
  location                 GEOGRAPHY(POINT, 4326) NOT NULL,  -- PostGIS
  location_address         TEXT,
  location_city            VARCHAR(100),
  location_state           VARCHAR(50),
  location_zip             VARCHAR(20),
  rating                   NUMERIC(3, 2) DEFAULT 0.00,  -- 0.00 to 5.00
  review_count             INTEGER DEFAULT 0,
  total_bookings           INTEGER DEFAULT 0,
  total_hours              NUMERIC(10, 2) DEFAULT 0.00,
  cancellation_count       INTEGER DEFAULT 0,
  repeat_booking_rate      NUMERIC(5, 2) DEFAULT 0.00,  -- percentage
  stripe_connect_id        VARCHAR(100) UNIQUE,  -- Stripe Connect Account ID
  stripe_onboarding_complete BOOLEAN DEFAULT FALSE,
  background_check_verified BOOLEAN DEFAULT FALSE,
  insurance_verified       BOOLEAN DEFAULT FALSE,
  status                   cleaner_status NOT NULL DEFAULT 'pending',
  created_at               TIMESTAMPTZ DEFAULT NOW(),
  updated_at               TIMESTAMPTZ DEFAULT NOW()
);

-- Geospatial index for location-based search
CREATE INDEX idx_cleaners_location ON cleaners USING GIST(location);
CREATE INDEX idx_cleaners_rating ON cleaners(rating DESC);
CREATE INDEX idx_cleaners_status ON cleaners(status);
CREATE INDEX idx_cleaners_stripe_connect ON cleaners(stripe_connect_id);
```

**Notes:**
- `location` uses PostGIS GEOGRAPHY type for accurate distance calculations
- `base_rate` in cents to avoid floating-point issues
- `stripe_connect_id` for Stripe Connect payouts

---

### cleaner_services

**Services offered by each cleaner with optional price add-ons.**

```sql
CREATE TYPE service_type AS ENUM (
  'standard',
  'deep_clean',
  'move_in_out',
  'inside_fridge',
  'inside_oven',
  'inside_cabinets',
  'laundry',
  'windows'
);

CREATE TABLE cleaner_services (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cleaner_id   UUID NOT NULL REFERENCES cleaners(user_id) ON DELETE CASCADE,
  service_type service_type NOT NULL,
  price_addon  INTEGER NOT NULL DEFAULT 0,  -- cents, can be 0
  created_at   TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(cleaner_id, service_type)
);

CREATE INDEX idx_cleaner_services_cleaner ON cleaner_services(cleaner_id);
CREATE INDEX idx_cleaner_services_type ON cleaner_services(service_type);
```

---

### availability_rules

**Weekly recurring availability patterns.**

```sql
CREATE TYPE day_of_week AS ENUM (
  'monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday'
);

CREATE TABLE availability_rules (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cleaner_id UUID NOT NULL REFERENCES cleaners(user_id) ON DELETE CASCADE,
  day        day_of_week NOT NULL,
  start_time TIME NOT NULL,
  end_time   TIME NOT NULL,
  timezone   VARCHAR(50) NOT NULL DEFAULT 'America/Los_Angeles',
  created_at TIMESTAMPTZ DEFAULT NOW(),

  CHECK (end_time > start_time)
);

CREATE INDEX idx_availability_cleaner ON availability_rules(cleaner_id);
CREATE INDEX idx_availability_day ON availability_rules(day);
```

**Alternative: JSONB approach for flexibility**
```sql
-- Single row per cleaner with JSONB
CREATE TABLE availability_rules_v2 (
  cleaner_id UUID PRIMARY KEY REFERENCES cleaners(user_id) ON DELETE CASCADE,
  timezone   VARCHAR(50) NOT NULL DEFAULT 'America/Los_Angeles',
  weekly_schedule JSONB NOT NULL,  -- { "monday": [{ "start": "09:00", "end": "17:00" }], ... }
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

### availability_blackouts

**One-time unavailable periods (vacations, holidays).**

```sql
CREATE TABLE availability_blackouts (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cleaner_id UUID NOT NULL REFERENCES cleaners(user_id) ON DELETE CASCADE,
  start_time TIMESTAMPTZ NOT NULL,
  end_time   TIMESTAMPTZ NOT NULL,
  reason     TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),

  CHECK (end_time > start_time)
);

CREATE INDEX idx_blackouts_cleaner ON availability_blackouts(cleaner_id);
CREATE INDEX idx_blackouts_time_range ON availability_blackouts USING GIST(
  tstzrange(start_time, end_time)
);
```

---

## Booking Tables

### booking_holds

**Temporary holds during checkout (5-minute expiry).**

```sql
CREATE TABLE booking_holds (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cleaner_id       UUID NOT NULL REFERENCES cleaners(user_id),
  customer_id      UUID NOT NULL REFERENCES users(id),
  start_time       TIMESTAMPTZ NOT NULL,
  end_time         TIMESTAMPTZ NOT NULL,
  expires_at       TIMESTAMPTZ NOT NULL,
  idempotency_key  VARCHAR(255) NOT NULL UNIQUE,
  created_at       TIMESTAMPTZ DEFAULT NOW(),

  CHECK (end_time > start_time),
  CHECK (expires_at > created_at)
);

-- Critical: Prevent overlapping holds
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE booking_holds
  ADD CONSTRAINT no_overlap_holds
  EXCLUDE USING GIST (
    cleaner_id WITH =,
    tstzrange(start_time, end_time) WITH &&
  );

CREATE INDEX idx_holds_cleaner ON booking_holds(cleaner_id);
CREATE INDEX idx_holds_expires ON booking_holds(expires_at);
CREATE INDEX idx_holds_idempotency ON booking_holds(idempotency_key);
```

**Cleanup Job:**
```sql
-- Run every minute to clean up expired holds
DELETE FROM booking_holds WHERE expires_at < NOW();
```

The exclusion constraint intentionally applies to all holds. Expired holds are released by the cleanup job, and booking reads must always filter on `expires_at > NOW()`.

---

### bookings

**Confirmed bookings.**

```sql
CREATE TYPE booking_status AS ENUM (
  'confirmed',
  'in_progress',
  'completed',
  'cancelled_by_customer',
  'cancelled_by_cleaner',
  'disputed'
);

CREATE TABLE bookings (
  id                   UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cleaner_id           UUID NOT NULL REFERENCES cleaners(user_id),
  customer_id          UUID NOT NULL REFERENCES users(id),
  start_time           TIMESTAMPTZ NOT NULL,
  end_time             TIMESTAMPTZ NOT NULL,
  status               booking_status NOT NULL DEFAULT 'confirmed',

  -- Pricing (all in cents)
  cleaner_rate         INTEGER NOT NULL,  -- rate at time of booking
  hours                NUMERIC(5, 2) NOT NULL,
  subtotal             INTEGER NOT NULL,  -- cleaner_rate × hours
  platform_fee         INTEGER NOT NULL,
  total_price          INTEGER NOT NULL,  -- subtotal + platform_fee

  -- Payment
  payment_intent_id    VARCHAR(100) UNIQUE,
  payment_status       VARCHAR(50),

  -- Completion
  completed_at         TIMESTAMPTZ,
  customer_confirmed   BOOLEAN DEFAULT FALSE,
  customer_confirmed_at TIMESTAMPTZ,

  -- Cancellation
  cancelled_at         TIMESTAMPTZ,
  cancellation_reason  TEXT,
  refund_amount        INTEGER,

  -- Location
  address_street       VARCHAR(255),
  address_unit         VARCHAR(50),
  address_city         VARCHAR(100) NOT NULL DEFAULT 'San Francisco',
  address_state        VARCHAR(50) NOT NULL DEFAULT 'CA',
  address_zip          VARCHAR(20) NOT NULL,
  address_location     GEOGRAPHY(POINT, 4326),  -- PostGIS for distance calc
  access_instructions  TEXT,  -- gate code, parking, etc.

  -- Metadata
  special_instructions TEXT,
  created_at           TIMESTAMPTZ DEFAULT NOW(),
  updated_at           TIMESTAMPTZ DEFAULT NOW(),

  CHECK (end_time > start_time),
  CHECK (total_price = subtotal + platform_fee)
);

-- Critical: Prevent double-booking
ALTER TABLE bookings
  ADD CONSTRAINT no_overlap_confirmed
  EXCLUDE USING GIST (
    cleaner_id WITH =,
    tstzrange(start_time, end_time) WITH &&
  )
  WHERE (status IN ('confirmed', 'in_progress', 'completed'));

CREATE INDEX idx_bookings_cleaner ON bookings(cleaner_id);
CREATE INDEX idx_bookings_customer ON bookings(customer_id);
CREATE INDEX idx_bookings_status ON bookings(status);
CREATE INDEX idx_bookings_start_time ON bookings(start_time DESC);
CREATE INDEX idx_bookings_payment_intent ON bookings(payment_intent_id);
CREATE INDEX idx_bookings_created ON bookings(created_at DESC);
CREATE INDEX idx_bookings_address_zip ON bookings(address_zip);
```

---

### booking_services

**Selected services for each booking.**

```sql
CREATE TABLE booking_services (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
  service_type service_type NOT NULL,
  price_addon INTEGER NOT NULL DEFAULT 0,

  UNIQUE(booking_id, service_type)
);

CREATE INDEX idx_booking_services_booking ON booking_services(booking_id);
```

---

## Payment Tables

### payments

**Payment records (linked to Stripe PaymentIntent).**

```sql
CREATE TYPE payment_status AS ENUM (
  'pending',
  'processing',
  'succeeded',
  'failed',
  'cancelled'
);

CREATE TABLE payments (
  id                     UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id             UUID UNIQUE REFERENCES bookings(id),
  stripe_payment_intent_id VARCHAR(100) NOT NULL UNIQUE,
  amount                 INTEGER NOT NULL,
  currency               VARCHAR(3) DEFAULT 'usd',
  status                 payment_status NOT NULL DEFAULT 'pending',
  failure_reason         TEXT,
  created_at             TIMESTAMPTZ DEFAULT NOW(),
  updated_at             TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_payments_booking ON payments(booking_id);
CREATE INDEX idx_payments_stripe_intent ON payments(stripe_payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
```

---

### payouts

**Cleaner payouts (Stripe Connect transfers).**

```sql
CREATE TYPE payout_status AS ENUM (
  'pending',
  'processing',
  'succeeded',
  'failed',
  'cancelled'
);

CREATE TABLE payouts (
  id                    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id            UUID UNIQUE REFERENCES bookings(id),
  cleaner_id            UUID NOT NULL REFERENCES cleaners(user_id),
  amount                INTEGER NOT NULL,
  currency              VARCHAR(3) DEFAULT 'usd',
  stripe_transfer_id    VARCHAR(100) UNIQUE,
  status                payout_status NOT NULL DEFAULT 'pending',
  failure_reason        TEXT,
  retry_count           INTEGER DEFAULT 0,
  scheduled_at          TIMESTAMPTZ NOT NULL,
  processed_at          TIMESTAMPTZ,
  created_at            TIMESTAMPTZ DEFAULT NOW(),
  updated_at            TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_payouts_booking ON payouts(booking_id);
CREATE INDEX idx_payouts_cleaner ON payouts(cleaner_id);
CREATE INDEX idx_payouts_status ON payouts(status);
CREATE INDEX idx_payouts_scheduled ON payouts(scheduled_at);
CREATE INDEX idx_payouts_stripe_transfer ON payouts(stripe_transfer_id);
```

---

### refunds

**Refund records.**

```sql
CREATE TABLE refunds (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id          UUID REFERENCES bookings(id),
  payment_id          UUID REFERENCES payments(id),
  stripe_refund_id    VARCHAR(100) UNIQUE,
  amount              INTEGER NOT NULL,
  reason              TEXT,
  status              VARCHAR(50) NOT NULL,
  created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_refunds_booking ON refunds(booking_id);
CREATE INDEX idx_refunds_payment ON refunds(payment_id);
```

---

## Review & Reputation Tables

### reviews

**Customer reviews of cleaners.**

```sql
CREATE TABLE reviews (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id   UUID UNIQUE REFERENCES bookings(id) ON DELETE CASCADE,
  cleaner_id   UUID NOT NULL REFERENCES cleaners(user_id),
  customer_id  UUID NOT NULL REFERENCES users(id),
  rating       INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
  text         TEXT,
  response     TEXT,  -- Cleaner's response (optional)
  responded_at TIMESTAMPTZ,
  is_verified  BOOLEAN DEFAULT TRUE,  -- Verified booking
  is_flagged   BOOLEAN DEFAULT FALSE,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_reviews_cleaner ON reviews(cleaner_id);
CREATE INDEX idx_reviews_customer ON reviews(customer_id);
CREATE INDEX idx_reviews_booking ON reviews(booking_id);
CREATE INDEX idx_reviews_rating ON reviews(rating DESC);
CREATE INDEX idx_reviews_created ON reviews(created_at DESC);
```

**Trigger: Update cleaner rating on new review**
```sql
CREATE OR REPLACE FUNCTION update_cleaner_rating()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE cleaners
  SET
    rating = (SELECT AVG(rating) FROM reviews WHERE cleaner_id = NEW.cleaner_id),
    review_count = (SELECT COUNT(*) FROM reviews WHERE cleaner_id = NEW.cleaner_id),
    updated_at = NOW()
  WHERE user_id = NEW.cleaner_id;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_cleaner_rating
AFTER INSERT OR UPDATE ON reviews
FOR EACH ROW
EXECUTE FUNCTION update_cleaner_rating();
```

---

### review_reports

**Flagged reviews for moderation.**

```sql
CREATE TABLE review_reports (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  review_id  UUID REFERENCES reviews(id) ON DELETE CASCADE,
  reporter_id UUID REFERENCES users(id),
  reason     TEXT NOT NULL,
  status     VARCHAR(50) DEFAULT 'pending',  -- pending, reviewed, dismissed
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_review_reports_review ON review_reports(review_id);
CREATE INDEX idx_review_reports_status ON review_reports(status);
```

---

## Bonus System Tables

### bonus_periods

**Monthly/yearly bonus periods.**

```sql
CREATE TYPE bonus_period_type AS ENUM ('monthly', 'yearly');
CREATE TYPE bonus_period_status AS ENUM ('pending', 'calculating', 'distributed', 'cancelled');

CREATE TABLE bonus_periods (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  type         bonus_period_type NOT NULL,
  year         INTEGER NOT NULL,
  month        INTEGER,  -- NULL for yearly
  gmv          INTEGER NOT NULL,  -- Total GMV for period (cents)
  pool_amount  INTEGER NOT NULL,  -- Total bonus pool (cents)
  status       bonus_period_status NOT NULL DEFAULT 'pending',
  calculated_at TIMESTAMPTZ,
  distributed_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(type, year, month)
);

CREATE INDEX idx_bonus_periods_type_year_month ON bonus_periods(type, year, month);
CREATE INDEX idx_bonus_periods_status ON bonus_periods(status);
```

---

### bonus_awards

**Individual bonus distributions to cleaners.**

```sql
CREATE TABLE bonus_awards (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bonus_period_id  UUID REFERENCES bonus_periods(id) ON DELETE CASCADE,
  cleaner_id       UUID REFERENCES cleaners(user_id),
  score            NUMERIC(10, 2) NOT NULL,
  amount           INTEGER NOT NULL,  -- cents
  stripe_transfer_id VARCHAR(100) UNIQUE,
  status           VARCHAR(50) DEFAULT 'pending',
  paid_at          TIMESTAMPTZ,
  created_at       TIMESTAMPTZ DEFAULT NOW(),

  UNIQUE(bonus_period_id, cleaner_id)
);

CREATE INDEX idx_bonus_awards_period ON bonus_awards(bonus_period_id);
CREATE INDEX idx_bonus_awards_cleaner ON bonus_awards(cleaner_id);
CREATE INDEX idx_bonus_awards_status ON bonus_awards(status);
```

---

## Admin & Support Tables

### audit_logs

**All state-changing actions for compliance and debugging.**

```sql
CREATE TABLE audit_logs (
  id         BIGSERIAL PRIMARY KEY,
  actor_id   UUID REFERENCES users(id),
  action     VARCHAR(100) NOT NULL,  -- e.g., 'booking.created', 'user.suspended'
  entity_type VARCHAR(50) NOT NULL,   -- e.g., 'booking', 'user', 'payout'
  entity_id  UUID NOT NULL,
  metadata   JSONB,                   -- Additional context
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_actor ON audit_logs(actor_id);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created ON audit_logs(created_at DESC);

-- Partition by month for scalability
CREATE TABLE audit_logs_2024_01 PARTITION OF audit_logs
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

### disputes

**Customer disputes and resolutions.**

```sql
CREATE TYPE dispute_status AS ENUM (
  'open',
  'investigating',
  'resolved_refund',
  'resolved_reclean',
  'resolved_dismiss',
  'escalated'
);

CREATE TABLE disputes (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  booking_id   UUID REFERENCES bookings(id),
  customer_id  UUID REFERENCES users(id),
  cleaner_id   UUID REFERENCES cleaners(user_id),
  reason       TEXT NOT NULL,
  status       dispute_status NOT NULL DEFAULT 'open',
  resolution   TEXT,
  refund_amount INTEGER,
  admin_notes  TEXT,
  resolved_by  UUID REFERENCES users(id),
  resolved_at  TIMESTAMPTZ,
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_disputes_booking ON disputes(booking_id);
CREATE INDEX idx_disputes_customer ON disputes(customer_id);
CREATE INDEX idx_disputes_cleaner ON disputes(cleaner_id);
CREATE INDEX idx_disputes_status ON disputes(status);
```

---

### support_tickets

**General support requests.**

```sql
CREATE TYPE ticket_status AS ENUM ('open', 'in_progress', 'resolved', 'closed');
CREATE TYPE ticket_priority AS ENUM ('low', 'medium', 'high', 'urgent');

CREATE TABLE support_tickets (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID REFERENCES users(id),
  subject     VARCHAR(255) NOT NULL,
  description TEXT NOT NULL,
  status      ticket_status NOT NULL DEFAULT 'open',
  priority    ticket_priority NOT NULL DEFAULT 'medium',
  assigned_to UUID REFERENCES users(id),
  resolved_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_tickets_user ON support_tickets(user_id);
CREATE INDEX idx_tickets_status ON support_tickets(status);
CREATE INDEX idx_tickets_assigned ON support_tickets(assigned_to);
```

---

### webhook_events

**Stripe webhook deduplication.**

```sql
CREATE TABLE webhook_events (
  id           VARCHAR(100) PRIMARY KEY,  -- Stripe event ID
  type         VARCHAR(100) NOT NULL,
  data         JSONB NOT NULL,
  processed    BOOLEAN DEFAULT FALSE,
  processed_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_webhook_events_type ON webhook_events(type);
CREATE INDEX idx_webhook_events_processed ON webhook_events(processed);
CREATE INDEX idx_webhook_events_created ON webhook_events(created_at DESC);
```

---

### api_idempotency_keys

**Cross-endpoint API idempotency store for mutating requests.**

```sql
CREATE TABLE api_idempotency_keys (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  idempotency_key VARCHAR(100) NOT NULL,
  route           VARCHAR(255) NOT NULL,
  method          VARCHAR(10) NOT NULL,
  request_hash    VARCHAR(64) NOT NULL,
  response_status INTEGER NOT NULL,
  response_body   JSONB NOT NULL,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  expires_at      TIMESTAMPTZ NOT NULL,
  UNIQUE(idempotency_key, route, method)
);

CREATE INDEX idx_idempotency_keys_expires ON api_idempotency_keys(expires_at);
CREATE INDEX idx_idempotency_keys_route_method ON api_idempotency_keys(route, method);
```

**Cleanup Job:**
```sql
-- Run every hour to remove expired idempotency records
DELETE FROM api_idempotency_keys WHERE expires_at < NOW();
```

Use this store for mutating endpoints such as booking confirmation/cancellation/dispute and payment-intent creation. Continue using `webhook_events.id` for Stripe event deduplication.

---

## Indexes Strategy Summary

### Primary Indexes (created above)
- **Foreign keys** - All FK columns indexed
- **Geospatial** - `cleaners.location` (GIST)
- **Time ranges** - `booking_holds`, `bookings` (GIST + exclusion)
- **Search** - `users.email`, `users.phone`, rating, status

### Composite Indexes (for common queries)
```sql
-- Cleaner search (geo + rating + status)
CREATE INDEX idx_cleaners_search
ON cleaners(status, rating DESC)
WHERE status = 'active';

-- Booking queries by cleaner + time
CREATE INDEX idx_bookings_cleaner_time
ON bookings(cleaner_id, start_time DESC);

-- Customer booking history
CREATE INDEX idx_bookings_customer_created
ON bookings(customer_id, created_at DESC);

-- Upcoming bookings
CREATE INDEX idx_bookings_upcoming
ON bookings(start_time ASC)
WHERE status IN ('confirmed', 'in_progress');
```

---

## Data Integrity Rules

### Constraints Summary
- **No overlapping bookings** - Exclusion constraint on `bookings`
- **No overlapping holds** - Exclusion constraint on `booking_holds`
- **Rating range** - CHECK rating BETWEEN 1 AND 5
- **Time logic** - CHECK end_time > start_time
- **Price logic** - CHECK total_price = subtotal + platform_fee
- **Unique constraints** - Prevent duplicate reviews, services per cleaner

---

## Data Migration & Seeding

### Initial Seed Data
```sql
-- Insert admin user
INSERT INTO users (email, role, status, first_name, last_name)
VALUES ('admin@yourdomain.com', 'admin', 'active', 'Admin', 'User');

-- Insert test cleaner
INSERT INTO users (email, role, status, first_name, last_name)
VALUES ('cleaner@example.com', 'cleaner', 'active', 'Jane', 'Doe')
RETURNING id;

INSERT INTO cleaners (user_id, bio, base_rate, location)
VALUES (
  '<user_id_from_above>',
  'Professional cleaner with 5 years experience',
  5500,  -- $55/hr
  ST_SetSRID(ST_MakePoint(-122.4194, 37.7749), 4326)  -- SF coordinates
);
```

---

## Performance Optimizations

### Partitioning Strategy (Future)
```sql
-- Partition bookings by month (when >1M records)
CREATE TABLE bookings (
  -- columns...
) PARTITION BY RANGE (start_time);

CREATE TABLE bookings_2024_01 PARTITION OF bookings
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE bookings_2024_02 PARTITION OF bookings
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

### Materialized Views (for heavy aggregations)
```sql
-- Top cleaners leaderboard (refresh daily)
CREATE MATERIALIZED VIEW top_cleaners AS
SELECT
  c.user_id,
  c.rating,
  c.review_count,
  c.total_bookings,
  c.repeat_booking_rate,
  (c.rating * 50 + c.repeat_booking_rate * 30 + c.total_hours * 0.2 - c.cancellation_count * 20) AS score
FROM cleaners c
WHERE c.status = 'active'
ORDER BY score DESC
LIMIT 100;

CREATE UNIQUE INDEX ON top_cleaners(user_id);

-- Refresh nightly
REFRESH MATERIALIZED VIEW CONCURRENTLY top_cleaners;
```

---

## Backup & Recovery

### Backup Strategy
- **Full backup** - Daily at 2 AM
- **Incremental backup** - Every 6 hours
- **WAL archiving** - Continuous (point-in-time recovery)
- **Retention** - 30 days

### RDS Automated Backups
```terraform
resource "aws_db_instance" "main" {
  backup_retention_period = 30
  backup_window          = "02:00-03:00"
  maintenance_window     = "sun:03:00-sun:04:00"
}
```

---

## Monitoring Queries

### Long-running queries
```sql
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - pg_stat_activity.query_start > interval '5 seconds'
ORDER BY duration DESC;
```

### Table sizes
```sql
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Index usage
```sql
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan AS index_scans,
  idx_tup_read AS tuples_read,
  idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

---

## Security

### Row-Level Security (RLS)
```sql
-- Enable RLS on sensitive tables
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

-- Policy: Customers can only see their own bookings
CREATE POLICY customer_bookings ON bookings
  FOR SELECT
  USING (customer_id = current_setting('app.user_id')::UUID);

-- Policy: Cleaners can only see their own bookings
CREATE POLICY cleaner_bookings ON bookings
  FOR SELECT
  USING (cleaner_id = current_setting('app.user_id')::UUID);

-- Usage in app:
SET app.user_id = '<current_user_id>';
```

### Encryption
- **At rest** - RDS encryption enabled
- **In transit** - SSL/TLS required
- **PII** - Consider `pgcrypto` for sensitive fields

---

## Next Steps

See related docs:
- [Backend Architecture](./BACKEND_ARCHITECTURE.md)
- [API Specification](./API_SPECIFICATION.md)
- [Infrastructure & Deployment](./INFRASTRUCTURE_DEPLOYMENT.md)
