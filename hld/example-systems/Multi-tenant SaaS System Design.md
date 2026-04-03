# Multi-tenant SaaS System Design

## System Overview
A multi-tenant SaaS platform architecture where multiple customers (tenants) share the same infrastructure while having isolated data, configurations, and experiences — covering the three tenancy models, data isolation strategies, and operational concerns.

## Architecture Diagram
#### (To be inserted)

## 1. Requirements

### Functional Requirements
- Tenant onboarding and provisioning
- Tenant-specific configuration and customization
- Data isolation between tenants
- Tenant-specific feature flags and plan limits
- Usage metering and billing per tenant
- Admin portal for tenant management
- Tenant-specific SLAs

### Non-Functional Requirements
- Isolation: one tenant's data must never be accessible to another
- Availability: 99.9% per tenant (noisy neighbor protection)
- Scalability: support 10K+ tenants, from small (10 users) to enterprise (100K users)
- Security: tenant data encrypted at rest and in transit
- Compliance: GDPR, SOC2 — tenant data residency requirements

## 2. Tenancy Models

### Model 1 — Silo (Dedicated per Tenant)
Each tenant gets their own dedicated infrastructure (DB, compute, storage).

```
Tenant A: [App Server A] → [DB A]
Tenant B: [App Server B] → [DB B]
Tenant C: [App Server C] → [DB C]
```

**Pros:**
- Maximum isolation — no noisy neighbor
- Easy compliance (data residency, GDPR deletion)
- Tenant-specific scaling
- Simple data migration and backup per tenant

**Cons:**
- High cost — N tenants = N databases
- Operational overhead — manage N separate environments
- Resource waste — small tenants have underutilized dedicated infra

**Best for:** Enterprise/large tenants with strict compliance requirements

### Model 2 — Pool (Shared Everything)
All tenants share the same DB, differentiated by `tenant_id` column.

```
All Tenants → [Shared App Servers] → [Shared DB with tenant_id column]
```

**Pros:**
- Low cost — one DB for all tenants
- Easy to operate — single environment
- Efficient resource utilization

**Cons:**
- Noisy neighbor — one tenant's heavy query affects others
- Harder compliance — GDPR deletion requires deleting rows across many tables
- Security risk — a bug could expose cross-tenant data
- Harder to scale individual tenants

**Best for:** SMB/startup tenants with low compliance requirements

### Model 3 — Bridge (Shared Compute, Isolated DB)
Shared application servers but separate database per tenant (or schema per tenant).

```
All Tenants → [Shared App Servers] → [DB per tenant or schema per tenant]
```

**Pros:**
- Balance of cost and isolation
- Data isolation without dedicated compute
- Easier compliance than pool model
- Can scale DB independently per tenant

**Cons:**
- More complex connection pooling (N databases)
- Schema migrations must run across all tenant DBs
- More expensive than pool model

**Best for:** Mid-market tenants; most common SaaS architecture

## 3. Core Components

**API Gateway** — Auth, rate limiting per tenant, routing; extracts `tenantId` from JWT or subdomain

**Tenant Service** — Tenant CRUD, provisioning, plan management, feature flags

**Auth Service** — Multi-tenant auth; JWT contains `tenantId` + `userId`; validates tenant is active

**Application Services** — Business logic services; always scoped to `tenantId`; never cross-tenant queries

**Tenant Config Service** — Per-tenant configuration, customization, feature flags; cached in Redis

**Usage Metering Service** — Tracks API calls, storage, users per tenant; feeds billing

**Billing Service** — Subscription management, invoice generation, payment processing

**Notification Service** — Tenant-scoped notifications; respects tenant communication preferences

**Tenant DB Router** — Routes DB connections to correct tenant DB/schema (Bridge model)

**Control Plane DB (PostgreSQL)** — Tenant registry, plans, billing, config — shared across all tenants

**Tenant Data DB** — Per-tenant data (Bridge/Silo) or shared with tenant_id (Pool)

**Redis** — Tenant config cache, rate limiting per tenant, session store

**Kafka** — Usage events, billing events, async operations

## 4. Database Design

### Control Plane — tenants

| Field | Type |
|---|---|
| tenant_id | UUID (PK) |
| name | VARCHAR |
| subdomain | VARCHAR, unique |
| plan | ENUM (free / starter / pro / enterprise) |
| status | ENUM (active / suspended / churned) |
| db_connection_string | TEXT (Bridge/Silo model) |
| region | VARCHAR |
| created_at | TIMESTAMP |

### Control Plane — tenant_config

| Field | Type |
|---|---|
| tenant_id | UUID (PK) |
| feature_flags | JSONB |
| limits | JSONB (max_users, api_rate_limit, storage_gb) |
| custom_domain | VARCHAR, nullable |
| sso_config | JSONB, nullable |
| updated_at | TIMESTAMP |

### Control Plane — usage_metrics

| Field | Type |
|---|---|
| tenant_id | UUID |
| metric_type | VARCHAR (api_calls / storage_gb / active_users) |
| value | BIGINT |
| period | DATE |
| updated_at | TIMESTAMP |

### Tenant Data DB (Bridge model — per tenant schema)

Each tenant gets their own schema: `tenant_{tenantId}.users`, `tenant_{tenantId}.data`, etc.

Pool model: every table has `tenant_id` column as first column in composite PK and all indexes.

### Redis Keys

| Key Pattern | Type | Value | TTL |
|---|---|---|---|
| `tenant:config:{tenantId}` | String | config JSON | 300s |
| `tenant:rate:{tenantId}` | Counter | API calls in window | 60s |
| `session:{tenantId}:{sessionId}` | String | userId | 86400s |

## 5. Key Flows

### 5.1 Tenant Onboarding

1. New customer signs up → Tenant Service
2. Create tenant record in Control Plane DB
3. Provision tenant infrastructure based on plan:
   - Free/Starter (Pool): create schema in shared DB, seed with default data
   - Pro (Bridge): provision dedicated DB instance, run schema migrations
   - Enterprise (Silo): provision dedicated cluster, configure VPC, custom domain
4. Create default admin user for tenant
5. Send welcome email with setup instructions
6. Tenant config initialized with plan defaults

### 5.2 Request Routing & Tenant Identification

Every request must be scoped to a tenant:

**Subdomain-based:** `acme.saas.com` → `tenantId = acme`
**JWT-based:** JWT contains `{tenantId, userId, role}`
**API key:** API key maps to `tenantId` in Control Plane DB

1. Request arrives at API Gateway
2. Extract `tenantId` from subdomain / JWT / API key
3. Validate tenant is active (Redis cache → Control Plane DB)
4. Apply tenant-specific rate limits: `INCR tenant:rate:{tenantId}` in Redis
5. Inject `tenantId` into request context
6. All downstream services use `tenantId` for data access

### 5.3 Data Access with Tenant Isolation

**Pool model:**
```sql
-- ALWAYS include tenant_id in every query
SELECT * FROM users WHERE tenant_id = ? AND user_id = ?
-- Never: SELECT * FROM users WHERE user_id = ?  ← cross-tenant leak risk
```

**Bridge model:**
```
DB Router: SELECT db_connection FROM tenants WHERE tenant_id = ?
Connect to tenant-specific DB
Execute query without tenant_id filter (DB is already isolated)
```

**Silo model:**
```
Each tenant has dedicated connection pool
No tenant_id needed in queries
```

### 5.4 Feature Flags & Plan Limits

1. Request arrives → Tenant Config Service checks `tenant:config:{tenantId}` in Redis
2. Cache miss: fetch from Control Plane DB, populate Redis (TTL 5min)
3. Check feature flag: `config.feature_flags.advanced_analytics = true`
4. Check plan limit: `config.limits.max_users = 100`; current users = 95 → allow
5. If limit exceeded: return 429 with upgrade prompt

### 5.5 Usage Metering & Billing

1. Every API call publishes usage event to Kafka: `{tenantId, metric: api_call, timestamp}`
2. Usage Metering Service consumes events, aggregates per tenant per day
3. Writes to `usage_metrics` table in Control Plane DB
4. Billing Service reads usage at end of billing period
5. Generates invoice based on plan + overage
6. Charges payment method via Payment Gateway

### 5.6 Schema Migrations (Bridge/Pool Model)

Challenge: running migrations across 10K tenant DBs/schemas.

1. New migration created (e.g., add column to `users` table)
2. Migration Runner fetches all active tenant DB connections
3. Runs migration on each tenant DB sequentially or in parallel batches
4. Tracks migration status per tenant in Control Plane DB
5. On failure: mark tenant as migration-failed, alert ops, retry

**Zero-downtime migrations:**
- Expand-contract pattern: add new column (nullable), backfill, make required, drop old column
- Never drop columns in same migration as adding new ones

## 6. Key Interview Concepts

### Noisy Neighbor Problem
In Pool model, one tenant running heavy queries degrades performance for all others. Solutions:
- Query timeout per tenant
- Rate limiting at DB level (connection pool limits per tenant)
- Move heavy tenants to Bridge/Silo model
- Read replicas for reporting queries

### Cross-Tenant Data Leak Prevention
The most critical security concern. Solutions:
- Pool model: row-level security (PostgreSQL RLS) — DB enforces tenant_id filter automatically
- All queries must include `tenant_id` in WHERE clause — enforced by ORM/middleware
- Integration tests that verify cross-tenant isolation
- Regular security audits

### GDPR Right to Erasure
Tenant requests data deletion. Solutions:
- Silo/Bridge: drop tenant DB/schema — clean and complete
- Pool: delete all rows with `tenant_id = X` across all tables — complex, must be thorough
- Soft delete first, hard delete after verification
- Audit log of deletion for compliance

### Connection Pooling at Scale
Bridge model with 10K tenants = 10K DB connections. Solutions:
- PgBouncer or connection pooler per tenant group
- Lazy connection: only open connection when tenant is active
- Connection limits per tenant based on plan

### Tenant Isolation in Caching
Redis cache must be tenant-scoped. Always prefix keys with `tenantId`. Never cache data that could be served to wrong tenant. Separate Redis instances for enterprise tenants (Silo model).

## 7. Failure Scenarios

### Tenant DB Failure (Bridge/Silo)
- Impact: only that tenant affected; other tenants unaffected
- Recovery: promote replica; tenant sees brief unavailability
- Prevention: per-tenant DB replication; automated failover

### Shared DB Failure (Pool)
- Impact: all tenants affected
- Recovery: promote replica; all tenants recover together
- Prevention: high-availability DB cluster; this is the main risk of Pool model

### Migration Failure on Tenant DB
- Impact: that tenant's DB is in inconsistent state
- Recovery: rollback migration for that tenant; mark as failed; alert ops
- Prevention: test migrations on staging tenant first; idempotent migrations

### Noisy Tenant Overloading Shared Resources
- Detection: latency spikes for other tenants; one tenant consuming disproportionate resources
- Recovery: throttle offending tenant; move to dedicated infrastructure
- Prevention: per-tenant rate limits; resource quotas; monitoring per tenant

### Cross-Tenant Data Leak Bug
- Detection: security audit, penetration test, or customer report
- Recovery: immediate incident response; identify affected tenants; notify per GDPR requirements
- Prevention: row-level security; automated cross-tenant isolation tests; code review for all DB queries
