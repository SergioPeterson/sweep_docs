# System Design Overview - Cleaning Marketplace

## Product Vision

A two-sided marketplace connecting cleaners and customers in San Francisco, built on principles of transparency, fair economics, and cleaner empowerment.

### Core Differentiators
- **No membership fees** - Pay-per-booking only
- **True all-in pricing** - California SB 478 compliant
- **Cleaner-first economics** - Cleaners set rates, keep most of revenue
- **Transparent bonuses** - Funded from dedicated GMV pool

### One-Line Pitch
*"A cleaner-first, no-membership booking platform with all-in upfront pricing—and bonuses funded transparently from a small fee pool."*

---

## Business Model

### Revenue Structure (Option A - Recommended)
- Customer pays: `Cleaner Rate + 8% Platform Fee` (all-in, displayed upfront)
- Cleaner receives: Full hourly rate × hours
- Revenue breakdown:
  - 1.5% GMV → Bonus pool
  - 2.5% → Payment processing
  - 4% → Operations, support, infrastructure

**Example Booking:**
```
Cleaner rate: $55/hr × 4 hours = $220
Platform fee (8%): $17.60
Total charged: $237.60

Revenue allocation:
- Bonus pool (1.5% of $220): $3.30
- Processing (~2.5%): $5.94
- Net revenue: $8.36
```

### Bonus System
**Top Cleaner Formula:**
```
Score = (Rating × 50) + (Repeat% × 30) + (Completed hours × 0.2) - (Cancellations × 20)
```

**Criteria:**
- Quality: Average rating (minimum 4.5/5)
- Reliability: Low cancellation rate
- Volume: Completed jobs/hours
- Customer love: Repeat booking rate

---

## Goals and Non-Goals

### Core Goals
1. **Low-latency browse/search** - Sub-200ms cleaner list/availability
2. **Strong consistency** - "Book once" guarantees (no double-booking)
3. **Secure payments** - PCI-compliant, fraud-resistant
4. **High availability** - 99.9% uptime, graceful degradation
5. **Fast iteration** - Ship features weekly, scale incrementally

### Non-Goals (Initially)
- Full microservices architecture
- Real-time location tracking
- Custom payment rails
- Multi-city support (SF-only initially)
- Mobile apps (web-first launch)

---

## System Architecture Philosophy

### Modular Monolith Approach

Start with a **single backend codebase** organized into **domain modules**, then extract services as needed.

**Why this works:**
- Faster development (no distributed system complexity)
- Easier debugging and testing
- Simple deployment pipeline
- Lower infrastructure costs
- Clear extraction path when needed

**When to extract services:**
1. Search/Discovery (when >10K cleaners)
2. Notifications (when >100K notifications/day)
3. Payment webhooks (for isolation and retry logic)

---

## High-Level System Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Clients                              │
│              (Web / Mobile Apps)                        │
└────────────────┬────────────────────────────────────────┘
                 │
         ┌───────▼────────┐
         │   CDN / Edge   │ (CloudFront/Vercel)
         │  Static Assets │
         └───────┬────────┘
                 │
         ┌───────▼────────┐
         │ Load Balancer  │ (ALB)
         │  / API Gateway │
         └───────┬────────┘
                 │
    ┌────────────▼─────────────┐
    │   Backend API Service    │
    │   (Modular Monolith)     │
    │                          │
    │  ┌─────────────────┐    │
    │  │ Auth & Identity │    │
    │  │ Cleaner Profile │    │
    │  │ Search          │    │
    │  │ Booking         │    │
    │  │ Payments        │    │
    │  │ Reviews         │    │
    │  │ Bonuses         │    │
    │  │ Admin           │    │
    │  └─────────────────┘    │
    └──┬──┬──┬──┬──┬──┬──┬───┘
       │  │  │  │  │  │  │
   ┌───▼──▼──▼──▼──▼──▼──▼────────────────┐
   │        Infrastructure Layer           │
   ├───────────────────────────────────────┤
   │  PostgreSQL  │  Redis   │  S3         │
   │  (Primary)   │  (Cache) │  (Storage)  │
   ├──────────────┼──────────┼─────────────┤
   │  SQS/Queue   │  Search  │  Stripe     │
   │  (Jobs)      │  (Geo)   │  (Payments) │
   ├──────────────┼──────────┼─────────────┤
   │  Email/SMS   │  Push    │  Analytics  │
   │  (Notif)     │  (Mobile)│  (Metrics)  │
   └──────────────┴──────────┴─────────────┘
```

---

## Core User Flows

### 1. Cleaner Onboarding
```
Create Account → Verify Phone/Email → Set Profile
→ Configure Pricing → Set Availability → Connect Stripe
→ Optional: Background Check → Active
```

### 2. Customer Booking
```
Search Cleaners (geo + filters) → View Cleaner Profile
→ Select Date/Time → Choose Services → Create Hold
→ Payment (Stripe) → Confirm Booking → Notifications Sent
```

### 3. Job Completion & Payout
```
Job Completed → Customer Confirms (or auto-confirm after 24h)
→ Release Payout → Stripe Transfer → Customer Reviews
→ Update Cleaner Score
```

### 4. Dispute & Resolution
```
Customer Reports Issue → Admin Review → Decision
→ (Refund Credit / Re-clean / Dismiss) → Update Records
→ Notify Parties
```

---

## Performance Targets

### Latency (p95)
- Search: <200ms
- Booking creation: <500ms
- Payment confirmation: <1s
- Profile page load: <300ms

### Throughput (MVP)
- 100 concurrent users
- 500 bookings/day
- 50 cleaners initially

### Throughput (Scale - Year 1)
- 1,000 concurrent users
- 10,000 bookings/day
- 500+ cleaners

### Availability
- API: 99.9% uptime
- Database: 99.95% (RDS Multi-AZ)
- Payments: 99.99% (Stripe SLA)

---

## Data Consistency Strategy

### Strong Consistency (ACID required)
- Booking creation/confirmation
- Payment transactions
- Payout records
- Availability conflicts

### Eventual Consistency (acceptable)
- Search index updates
- Notification delivery
- Analytics/metrics
- Bonus calculations

### Caching Strategy
- **Cache-aside** for read-heavy data (profiles, reviews)
- **Write-through** for critical counters (booking counts)
- **TTL-based** for search results (5-10 minutes)

---

## Security & Compliance

### Authentication & Authorization
- JWT-based auth (short-lived access tokens)
- Role-based access control (Customer, Cleaner, Admin)
- Multi-factor authentication (optional, for high-value accounts)

### Data Protection
- Encryption at rest (RDS, S3)
- Encryption in transit (TLS 1.3)
- PII pseudonymization in logs
- Signed URLs for user uploads

### Compliance Requirements
- **California SB 478** - All-in pricing displayed upfront
- **PCI DSS** - No card data stored (Stripe handles)
- **CCPA** - User data deletion on request
- **Worker Classification** - Cleaners control pricing/schedule (AB 5 mitigation)

### Audit & Monitoring
- All state-changing actions logged (`audit_log` table)
- Real-time fraud detection (abnormal booking patterns)
- Admin action tracking
- Payment reconciliation (daily)

---

## Scalability Roadmap

### Phase 1: Launch (0-500 bookings/day)
**Architecture:** Single-region modular monolith
- 1 API service (ECS/Fargate)
- 1 Worker service (SQS processing)
- PostgreSQL RDS (db.t4g.medium)
- Redis ElastiCache (cache.t4g.micro)

**Cost:** ~$500-800/month

### Phase 2: Growth (500-5,000 bookings/day)
**Architecture:** Scaled monolith with read replicas
- 2-4 API instances (auto-scaling)
- 2 Worker instances
- PostgreSQL read replica
- Redis cluster
- OpenSearch for search (optional)

**Cost:** ~$2,000-3,500/month

### Phase 3: Scale (5,000+ bookings/day)
**Architecture:** Hybrid (monolith + extracted services)
- Core monolith (booking, payments)
- Search service (dedicated)
- Notification service (dedicated)
- Multi-AZ deployment
- Database partitioning (bookings by month)

**Cost:** ~$5,000-10,000/month

---

## Technology Decisions Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| **Frontend** | Next.js + TypeScript | Fast iteration, SSR, great DX |
| **Backend** | Bun + Elysia | TypeScript end-to-end, fast runtime, modular structure |
| **Database** | PostgreSQL | ACID, PostGIS, mature tooling |
| **Cache** | Redis | Fast, simple, proven |
| **Queue** | SQS | Managed, reliable, scales automatically |
| **Search** | Postgres → OpenSearch | Start simple, upgrade when needed |
| **Payments** | Stripe | Industry standard, Connect for payouts |
| **Storage** | S3 | Cheap, durable, CDN-friendly |
| **Hosting** | AWS (ECS/Fargate) | Full control, easy scaling |
| **CDN** | CloudFront | AWS-native, good performance |
| **Observability** | OpenTelemetry + Sentry | Standard, vendor-neutral |

---

## Risk Mitigation

### Technical Risks

| Risk | Mitigation |
|------|-----------|
| **Double-booking** | Postgres exclusion constraints + idempotency |
| **Payment failures** | Webhook retries, manual reconciliation, customer support |
| **Search latency** | Caching, read replicas, eventual OpenSearch |
| **Database growth** | Partitioning, archiving, read replicas |
| **Third-party outages** | Circuit breakers, fallbacks, graceful degradation |

### Business Risks

| Risk | Mitigation |
|------|-----------|
| **Worker classification** | Cleaners set rates/schedule, legal review, audit trail |
| **Fraud (customer)** | Stripe Radar, device fingerprinting, pattern detection |
| **Fraud (cleaner)** | Review integrity checks, identity verification, background checks |
| **Supply shortage** | Incentive bonuses, low fees, fast payouts |
| **Demand shortage** | No-membership model, local SEO, partnerships |

---

## Development Principles

### Code Quality
- **TypeScript everywhere** - End-to-end type safety
- **Test coverage >80%** - Unit + integration tests
- **Code review required** - 2 approvals for critical paths
- **Linting enforced** - ESLint + Prettier

### Operational Excellence
- **Infrastructure as Code** - Terraform/CloudFormation
- **Automated deployments** - CI/CD (GitHub Actions)
- **Rollback ready** - Blue/green or canary deploys
- **Monitoring from day 1** - Metrics, logs, alerts

### Iteration Speed
- **Ship weekly** - Small, incremental releases
- **Feature flags** - Safe rollout, A/B testing
- **MVP mindset** - Build minimum, learn, iterate
- **Debt paydown** - 20% time for tech debt

---

## Success Metrics

### Product Metrics
- **Booking completion rate** - Target: >85%
- **Repeat booking rate** - Target: >40%
- **Cleaner retention** - Target: >70% after 3 months
- **Customer NPS** - Target: >50

### Technical Metrics
- **API error rate** - Target: <0.5%
- **p95 latency** - Target: <500ms
- **Deployment frequency** - Target: >2/week
- **MTTR (Mean Time to Recovery)** - Target: <30 minutes

### Business Metrics
- **GMV (Gross Merchandise Value)** - Track weekly
- **Take rate** - Target: 8%
- **Customer acquisition cost** - Target: <$30
- **Cleaner acquisition cost** - Target: <$100

---

## Next Steps

See detailed documentation:
- [Backend Architecture](./BACKEND_ARCHITECTURE.md)
- [Frontend Architecture](./FRONTEND_ARCHITECTURE.md)
- [Database Schema](./DATABASE_SCHEMA.md)
- [API Specification](./API_SPECIFICATION.md)
- [Infrastructure & Deployment](./INFRASTRUCTURE_DEPLOYMENT.md)
