# Wedding Rentals Marketplace Platform
## Architecture & Planning Document

**Date:** 2026-01-16
**Status:** Final Plan for Approval
**Target Market:** Bangalore (initial), India expansion, Global-ready
**Target Customer:** Couples planning their own weddings (B2C)
**Revenue Model:** Commission-based (10-20% per booking)
**Paradigm:** Data-First, ML-Ready, Analytics as First-Class Citizen

---

## 1. Executive Summary

This document outlines the architecture for a **wedding rentals marketplace** connecting vendors (decor, outfits, furniture, lighting, props) with **couples planning their own weddings**. Launching initially in **Bangalore** with a **commission-based revenue model** (10-20% per booking). This is **not a simple CRUD application** - it is a data-driven, two-sided marketplace where analytics and machine learning capabilities are foundational, not afterthoughts.

### Launch Strategy Decisions
- **Target Customer:** Couples planning their own weddings (B2C direct)
  - Simpler UX focused on discovery, trust, and ease of booking
  - One-time customers (optimize for conversion, not retention)
  - Mobile-first customer experience essential

- **Initial Market:** Bangalore
  - Tech-savvy customer base (high digital adoption)
  - Growing wedding market with less competition than Mumbai/Delhi
  - Smaller, manageable market for MVP validation
  - Strong vendor ecosystem

- **Revenue Model:** Commission-only (10-20% per booking)
  - No upfront costs for vendors (easier acquisition)
  - Aligns incentives (we earn when vendors earn)
  - Simple, transparent pricing

- **Vendor Mobile Strategy:** Native mobile app required for MVP
  - Vendors manage bookings from wedding venues (on-site management critical)
  - Push notifications for booking requests (reduce response time)
  - Offline capability for inventory checks
  - React Native for cross-platform (iOS + Android with shared codebase)

### Core Principles
- **Boring, proven architectures** over clever ones
- **Data capture from day one** - you can't retroactively collect data
- **Analytics-friendly schemas** - optimize for insights, not just transactions
- **Incremental ML adoption** - build foundations now, deploy models later
- **Single-city mastery** before multi-city expansion
- **Vendor-first experience** - vendors are the constrained resource

### Critical Success Factors
1. **Solve the chicken-and-egg problem**: Need vendors to attract customers, customers to attract vendors
2. **Prevent double-bookings**: Database-level constraints, not application hope
3. **Enable trust**: Reviews, verification, escrow payments
4. **Quality search/discovery**: Customers must find what they need easily
5. **Mobile vendor experience**: Vendors manage bookings from wedding venues
6. **ADDRESS CULTURAL STIGMA**: Overcome resistance to "reusing" wedding items through careful category positioning and messaging

---

## 2. Architecture Overview

### 2.1 High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Customer    │  │   Vendor     │  │    Admin     │      │
│  │  Web/Mobile  │  │  Dashboard   │  │   Portal     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ HTTPS/REST/GraphQL
                         │
┌────────────────────────▼────────────────────────────────────┐
│                    API GATEWAY                               │
│  - Auth (JWT)  - Rate Limiting  - Request Routing           │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼───────┐ ┌─────▼──────┐ ┌──────▼────────┐
│   BOOKING     │ │  INVENTORY │ │    VENDOR     │
│   SERVICE     │ │   SERVICE  │ │   SERVICE     │
└───────┬───────┘ └─────┬──────┘ └──────┬────────┘
        │               │               │
        │      ┌────────▼────────┐      │
        │      │  SEARCH SERVICE │      │
        │      │  (Algolia/ES)   │      │
        │      └─────────────────┘      │
        │                                │
┌───────▼────────────────────────────────▼────────┐
│         PRIMARY TRANSACTIONAL DATABASE          │
│              (PostgreSQL)                        │
│  - CRUD: vendors, items, users, profiles        │
│  - Event Sourced: bookings, payments            │
│  - Multi-tenant: Pool model (vendor_id)         │
└────────┬──────────────────────────────┬─────────┘
         │                              │
         │ Change Data Capture (CDC)    │
         │                              │
┌────────▼──────────────┐    ┌─────────▼──────────┐
│   EVENT STREAM        │    │   DATA WAREHOUSE   │
│   (Kafka/Kinesis)     │    │   (BigQuery/       │
│   - User events       │    │    Snowflake)      │
│   - Search queries    │───▶│   - Analytics      │
│   - Booking events    │    │   - ML training    │
│   - Price changes     │    │   - BI dashboards  │
└───────────────────────┘    └────────────────────┘
                                      │
                             ┌────────▼────────┐
                             │   ML PLATFORM   │
                             │  (Future Phase) │
                             │  - Forecasting  │
                             │  - Pricing      │
                             │  - Ranking      │
                             └─────────────────┘
```

### 2.2 Module Boundaries & Responsibilities

#### Core Services (MVP)

**1. Vendor Service**
- Vendor onboarding, profile management
- Verification workflows
- Subscription/tier management
- Performance analytics (response time, ratings)

**2. Inventory Service**
- Item catalog management (CRUD)
- Category/attribute hierarchies
- Pricing rules (base, seasonal, bulk discounts)
- Availability calendar management

**3. Booking Service** (Event Sourced)
- Booking request → confirmation workflow
- Double-booking prevention (DB constraints)
- State machine: requested → quoted → confirmed → fulfilled → returned
- Cancellation/refund logic

**4. Search Service** (Managed: Algolia initially)
- Geospatial search (vendors near venue)
- Faceted filtering (category, price, style, availability)
- Ranking algorithm (relevance + vendor quality)
- Auto-complete, typo tolerance

**5. Payment Service**
- Stripe Connect integration
- Split payments (vendor payout + platform commission)
- Escrow/delayed payout (hold until event completion)
- Refund processing

**6. User Service**
- Authentication (JWT-based)
- Customer profiles (wedding details, preferences)
- Role-based access control (customer, vendor, admin)

#### Analytics & Data Layer

**7. Event Ingestion Pipeline**
- Capture all user interactions (searches, views, favorites, cart adds)
- Booking lifecycle events
- Vendor interaction events
- Stream to data warehouse in real-time

**8. Analytics Service** (Read-Only Queries)
- Vendor dashboards (booking trends, revenue, popular items)
- Platform metrics (GMV, take rate, conversion funnels)
- Market intelligence (demand by category/city, seasonality)

**9. ML Platform** (Phase 2+)
- Feature store for model serving
- Training pipelines
- Model deployment infrastructure

---

## 3. Domain Model Design

### 3.1 Core Domains

#### Domain 1: Vendors

**Key Entities:**
- `vendors` - Business profiles, contact info, service areas
- `vendor_verifications` - Business license, insurance, ID verification status
- `vendor_subscriptions` - Tier (free, premium, enterprise), billing
- `vendor_performance_metrics` - Response time, completion rate, cancellation rate

**Relationships:**
- Vendor → owns → Inventory Items (1:N)
- Vendor → operates in → Cities (N:M via service areas)
- Vendor → receives → Bookings (1:N)

**Ownership Boundaries:**
- Vendors own their item catalog
- Platform owns vendor verification status
- Platform owns performance metrics (derived from bookings)

**High-Risk Assumptions:**
- **Assumption:** Vendors will accurately maintain availability calendars
  **Risk:** Manual updates → stale data → double bookings
  **Mitigation:** Automatic calendar sync, buffer inventory (never show 100% available)

- **Assumption:** Vendors respond to booking requests within SLA
  **Risk:** Slow responses → customer abandonment
  **Mitigation:** Track response time, downrank slow vendors in search

#### Domain 2: Inventory Items

**Key Entities:**
- `inventory_items` - Individual rental items (SKU-level)
- `item_categories` - Hierarchical taxonomy (Decor → Centerpieces → Floral)
- `item_attributes` - Metadata (color, style, material, dimensions)
- `pricing_rules` - Base price, seasonal multipliers, bulk discounts
- `item_availability` - Date-based availability tracking
- **NEW:** `item_condition` - Quality/newness tracking (brand_new, like_new, excellent, good)
- **NEW:** `cleaning_records` - Hygiene transparency (last_cleaned_at, certificate_url)

**Relationships:**
- Item → belongs to → Vendor (N:1)
- Item → categorized as → Category (N:1, potentially N:M with tagging)
- Item → booked via → Booking Line Items (1:N)

**Ownership Boundaries:**
- Vendors fully own item details (description, images, pricing)
- Platform defines category taxonomy
- Availability is shared ownership (vendor sets, platform validates against bookings)
- **NEW:** Platform enforces quality/hygiene standards (condition ratings, cleaning requirements)

**High-Risk Assumptions:**
- **Assumption:** Items can be modeled as fungible units (10 identical chairs)
  **Risk:** Some items are unique (custom mandap), others are fungible
  **Mitigation:** Support both serialized (unique ID per physical item) and pooled inventory

- **Assumption:** Pricing is relatively stable
  **Risk:** Dynamic event-based pricing (premium for peak wedding dates)
  **Mitigation:** Time-based pricing rules, surge pricing capability (ML later)

- **CULTURAL ASSUMPTION:** Customers accept rented items for weddings
  **Risk:** Stigma around "reused" wedding items, especially personal/intimate categories
  **Mitigation:** Focus MVP on low-stigma categories (furniture, decor, lighting), premium positioning, hygiene transparency

#### Domain 3: Categories & Attributes

**Key Entities:**
- `categories` - Decor, Furniture, Lighting, Outfits, Props
- `category_attributes` - Attributes relevant per category (e.g., Outfits → Size, Color)
- `attribute_values` - Enumerated values for filtering (Color → Red, Gold, Pink)
- **NEW:** `category_stigma_level` - Cultural acceptance rating (low/medium/high)

**Relationships:**
- Category → has → Child Categories (hierarchical)
- Category → defines → Required/Optional Attributes
- Item → values for → Attributes (structured metadata)

**Design for Analytics:**
- Track category popularity over time
- Identify underserved categories (high search, low supply)
- Seasonal category trends (lighting demand in winter weddings)
- **NEW:** Track conversion rates by stigma_level (validate cultural assumptions)

#### Domain 4: Cities & Service Areas

**Key Entities:**
- `cities` - Operational cities (Mumbai, Delhi, Bangalore)
- `service_areas` - Geospatial polygons or radius-based coverage
- `venue_locations` - Popular wedding venues (for recommendations)

**Relationships:**
- Vendor → serves → Service Areas (N:M)
- Booking → event at → City/Venue (N:1)

**Ownership Boundaries:**
- Platform defines operational cities
- Vendors define their service radius
- Customers specify event location (venue address or city)

**High-Risk Assumptions:**
- **Assumption:** Vendors serve entire city uniformly
  **Risk:** Some vendors only serve specific neighborhoods
  **Mitigation:** Support radius-based or polygon-based service areas

#### Domain 5: Bookings & Availability

**Key Entities (Event Sourced):**
- `bookings` - Aggregate root for booking workflow
- `booking_events` - Immutable event log (BookingRequested, QuoteProvided, Confirmed, etc.)
- `booking_line_items` - Items included in booking
- `booking_state` - Current state projection (pending, confirmed, fulfilled, cancelled)

**Relationships:**
- Booking → includes → Line Items → reference → Inventory Items
- Booking → made by → Customer (N:1)
- Booking → fulfilled by → Vendor (N:1, assuming single-vendor bookings initially)

**Ownership Boundaries:**
- Customer initiates booking
- Vendor confirms/rejects booking
- Platform enforces availability constraints
- Payment service handles financial transactions

**High-Risk Assumptions:**
- **Assumption:** Bookings are single-vendor
  **Risk:** Customers may want items from multiple vendors
  **Mitigation:** Phase 1 = single vendor, Phase 2 = multi-vendor cart with separate bookings per vendor

- **Assumption:** Date conflicts can be prevented at DB level
  **Risk:** Race conditions in high concurrency
  **Mitigation:** Database-level unique constraints on (item_id, date) for availability

#### Domain 6: Pricing Rules

**Key Entities:**
- `base_pricing` - Default daily/weekly rates
- `seasonal_pricing` - Date range multipliers (peak wedding season)
- `bulk_discounts` - Quantity-based discounts (rent 10+ chairs → 15% off)
- `dynamic_pricing_signals` - ML inputs (demand, competitor prices) - future

**Relationships:**
- Pricing Rule → applies to → Inventory Item
- Pricing Rule → valid during → Date Range
- Booking → calculates price using → Active Pricing Rules at booking time

**Design for Analytics:**
- Track price elasticity (how demand changes with price)
- A/B test pricing strategies
- Competitive pricing analysis (scrape competitor listings)

**High-Risk Assumptions:**
- **Assumption:** Price at booking time is honored (no repricing)
  **Risk:** Vendor raises price after quote
  **Mitigation:** Event sourcing captures QuoteProvided with locked price

---

### 3.2 Data Model Design Decisions

#### Multi-Tenancy: Pool Model (Shared Schema)

**Design:**
```sql
CREATE TABLE inventory_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vendor_id UUID NOT NULL REFERENCES vendors(id),
  category_id UUID NOT NULL REFERENCES categories(id),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  base_price_per_day DECIMAL(10,2),
  quantity_available INT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Critical for performance: index on tenant discriminator
CREATE INDEX idx_items_vendor ON inventory_items(vendor_id);
CREATE INDEX idx_items_category ON inventory_items(category_id);

-- Row-level security (PostgreSQL)
ALTER TABLE inventory_items ENABLE ROW LEVEL SECURITY;

CREATE POLICY vendor_isolation ON inventory_items
  USING (vendor_id = current_setting('app.current_vendor_id')::UUID);
```

**Rationale:**
- 80% of vendors fit standard offering → pool model most cost-effective
- Cross-vendor analytics trivial (no complex joins across databases)
- Reserve enterprise silo option for premium tier (dedicated DB instance)

**Security:**
- Application enforces vendor_id in all queries (JWT contains vendor_id)
- PostgreSQL RLS as defense-in-depth
- API gateway validates vendor access before routing requests

#### Event Sourcing: Selective Application

**Event Sourced Domains:**
1. **Bookings** (critical audit trail)
2. **Payments** (regulatory compliance)
3. **Pricing History** (honor price at booking time)

**Event Store Schema:**
```sql
CREATE TABLE booking_events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL,
  event_type VARCHAR(50) NOT NULL, -- BookingRequested, Confirmed, etc.
  event_data JSONB NOT NULL,
  event_version INT NOT NULL DEFAULT 1,
  occurred_at TIMESTAMPTZ DEFAULT NOW(),
  user_id UUID,

  -- Ordering guarantee within booking
  CONSTRAINT unique_booking_sequence UNIQUE (booking_id, occurred_at)
);

CREATE INDEX idx_events_booking ON booking_events(booking_id, occurred_at);
CREATE INDEX idx_events_type ON booking_events(event_type);
```

**Read Model Projections:**
```sql
-- Materialized view for queries (updated asynchronously)
CREATE MATERIALIZED VIEW booking_summary AS
SELECT
  booking_id,
  customer_id,
  vendor_id,
  event_date,
  total_amount,
  current_status, -- derived by replaying events
  created_at,
  last_updated_at
FROM booking_events_projection;

-- Refresh strategy: incremental updates via triggers or periodic refresh
```

**CRUD Domains:**
- Vendor profiles (current state sufficient)
- Inventory catalog (version history if needed, but not full event sourcing)
- User accounts
- Categories/attributes (mostly static)

#### Availability Conflict Prevention

**Day-Based Unique Constraint Approach:**
```sql
CREATE TABLE item_availability (
  item_id UUID NOT NULL REFERENCES inventory_items(id),
  rental_date DATE NOT NULL,
  quantity_reserved INT DEFAULT 0,
  quantity_available INT NOT NULL,

  -- Prevent double-booking at DB level
  PRIMARY KEY (item_id, rental_date),

  CONSTRAINT check_quantity CHECK (quantity_reserved <= quantity_available)
);

-- Booking attempt process:
-- 1. Start transaction
-- 2. SELECT FOR UPDATE on relevant dates
-- 3. Check quantity_available >= requested_quantity
-- 4. UPDATE quantity_reserved += requested_quantity
-- 5. Create booking_event
-- 6. Commit transaction
```

**Rationale:**
- Database guarantees no race conditions
- Works even under high concurrency
- Simple to reason about vs optimistic locking

**Buffer Strategy:**
- Never show items as available at 100% capacity
- Reserve 10% buffer for maintenance/cleaning time
- Example: 10 chairs available → show max 9 in UI

---

## 4. API Surface Plan

### 4.1 Core API Groups

#### Customer-Facing APIs

**Search & Discovery**
- `POST /api/v1/search` - Main search endpoint
  - Query params: location, category, date_range, price_range, style_filters
  - Returns: paginated items with availability, vendor info, pricing
  - Powered by: Algolia initially, Elasticsearch later

- `GET /api/v1/items/:id` - Item detail page
  - Returns: full item details, availability calendar, vendor profile, reviews

- `GET /api/v1/categories` - Category taxonomy
  - Hierarchical tree for navigation

**Booking Flow**
- `POST /api/v1/bookings/quote` - Request quote (pre-booking)
  - Input: items[], date_range, delivery_location
  - Returns: total_price, breakdown, availability_status

- `POST /api/v1/bookings` - Create booking
  - Input: quote_id, customer_info, payment_method
  - Returns: booking_id, status (pending_vendor_confirmation)
  - Triggers: BookingRequested event → vendor notification

- `GET /api/v1/bookings/:id` - Booking status
  - Returns: current state, timeline, vendor contact

- `POST /api/v1/bookings/:id/cancel` - Cancel booking
  - Triggers: BookingCancelled event, refund processing

**Reviews & Trust**
- `POST /api/v1/reviews` - Submit review (after event completion)
- `GET /api/v1/vendors/:id/reviews` - Vendor reviews

#### Vendor-Facing APIs

**Inventory Management**
- `POST /api/v1/vendor/items` - Create new item
- `PATCH /api/v1/vendor/items/:id` - Update item
- `DELETE /api/v1/vendor/items/:id` - Soft delete item
- `PUT /api/v1/vendor/items/:id/availability` - Update availability calendar

**Booking Management**
- `GET /api/v1/vendor/bookings` - List bookings (upcoming, pending, past)
- `POST /api/v1/vendor/bookings/:id/confirm` - Confirm booking request
- `POST /api/v1/vendor/bookings/:id/reject` - Reject with reason
- `POST /api/v1/vendor/bookings/:id/complete` - Mark fulfilled

**Analytics Dashboard**
- `GET /api/v1/vendor/analytics/overview` - Revenue, bookings, utilization
- `GET /api/v1/vendor/analytics/popular-items` - Top performers
- `GET /api/v1/vendor/analytics/calendar-heatmap` - Demand visualization

#### Analytics APIs (Internal/Admin)

**Market Intelligence**
- `GET /api/v1/analytics/demand-trends` - Category/city demand over time
- `GET /api/v1/analytics/supply-gaps` - Underserved categories
- `GET /api/v1/analytics/seasonality` - Booking patterns by month

**Vendor Performance**
- `GET /api/v1/analytics/vendor-rankings` - Vendor quality scores
- `GET /api/v1/analytics/response-times` - Vendor responsiveness

#### ML Inference APIs (Future)

**Recommendations**
- `GET /api/v1/recommendations/similar-items/:id` - Based on embeddings
- `GET /api/v1/recommendations/complete-the-look` - Bundle suggestions

**Pricing Optimization**
- `GET /api/v1/pricing/suggest` - ML-driven price recommendation for vendor

### 4.2 Versioning Strategy

**URL-Based Versioning:**
- `/api/v1/...` - Current stable version
- `/api/v2/...` - Major breaking changes (avoid when possible)

**Backward Compatibility:**
- Add new fields without removing old ones
- Use deprecation warnings (HTTP headers) before removing endpoints
- Maintain v1 for minimum 6 months after v2 launch

**GraphQL (Future Consideration):**
- Single endpoint: `/graphql`
- Schema evolution without versioning
- Useful for mobile apps (reduce over-fetching)

---

## 5. Data & Analytics Strategy

### 5.1 Data Capture: What & How

#### Raw Event Capture (Stream to Data Warehouse)

**User Behavior Events:**
```json
{
  "event_type": "item_viewed",
  "timestamp": "2026-01-16T10:30:00Z",
  "user_id": "uuid",
  "session_id": "uuid",
  "item_id": "uuid",
  "vendor_id": "uuid",
  "category": "decor/centerpieces",
  "search_context": {
    "query": "gold centerpieces",
    "filters": {"city": "Mumbai", "price_max": 5000},
    "position_in_results": 3
  },
  "device": "mobile_ios",
  "utm_source": "instagram"
}
```

**Booking Lifecycle Events:**
```json
{
  "event_type": "booking_confirmed",
  "timestamp": "2026-01-16T11:00:00Z",
  "booking_id": "uuid",
  "customer_id": "uuid",
  "vendor_id": "uuid",
  "event_date": "2026-02-14",
  "items": [
    {"item_id": "uuid", "quantity": 10, "unit_price": 500}
  ],
  "total_amount": 5000,
  "time_to_confirm_minutes": 45 // from request to vendor confirmation
}
```

**Search Query Events:**
```json
{
  "event_type": "search_performed",
  "timestamp": "2026-01-16T10:15:00Z",
  "user_id": "uuid",
  "query": "mandap decoration",
  "filters": {
    "city": "Delhi",
    "event_date": "2026-03-20",
    "budget": {"min": 10000, "max": 50000}
  },
  "results_count": 24,
  "clicked_items": ["uuid1", "uuid3"] // track which results clicked
}
```

#### Derived Data (Analytics Tables)

**Daily Aggregations:**
```sql
CREATE TABLE analytics_daily_item_performance (
  date DATE,
  item_id UUID,
  vendor_id UUID,
  category VARCHAR,

  -- Demand metrics
  view_count INT,
  favorite_count INT,
  quote_request_count INT,
  booking_count INT,

  -- Conversion funnel
  view_to_quote_rate DECIMAL(5,4),
  quote_to_booking_rate DECIMAL(5,4),

  -- Revenue
  revenue_generated DECIMAL(10,2),

  PRIMARY KEY (date, item_id)
);
```

**Vendor Performance Metrics:**
```sql
CREATE TABLE analytics_vendor_performance (
  vendor_id UUID,
  calculation_date DATE,

  -- Responsiveness
  avg_response_time_minutes DECIMAL(10,2),
  response_rate_pct DECIMAL(5,2), -- % of requests answered within SLA

  -- Quality
  avg_rating DECIMAL(3,2),
  review_count INT,
  completion_rate DECIMAL(5,2), -- % of bookings fulfilled without issues
  cancellation_rate DECIMAL(5,2),

  -- Business metrics
  total_bookings INT,
  total_revenue DECIMAL(10,2),
  utilization_rate DECIMAL(5,2), -- % of inventory booked

  PRIMARY KEY (vendor_id, calculation_date)
);
```

**Market Intelligence:**
```sql
CREATE TABLE analytics_demand_trends (
  date DATE,
  city VARCHAR,
  category VARCHAR,

  -- Demand signals
  search_volume INT,
  booking_volume INT,
  avg_booking_value DECIMAL(10,2),

  -- Supply
  active_vendors INT,
  available_items INT,

  -- Market health
  supply_demand_ratio DECIMAL(5,2), -- items per search
  avg_vendor_utilization DECIMAL(5,2),

  PRIMARY KEY (date, city, category)
);
```

### 5.2 Aggregation Granularity

**Real-Time (Streaming):**
- Current active users (dashboards)
- Inventory availability (booking conflicts)
- Fraud detection signals (suspicious booking patterns)

**Hourly:**
- Vendor dashboard metrics (bookings today, revenue this week)
- Search popularity trends

**Daily:**
- Item performance metrics
- Vendor performance scores
- Demand forecasting inputs

**Weekly:**
- Seasonality analysis
- Cohort retention (customers booking again)
- A/B test results

**Monthly:**
- Financial reporting (GMV, take rate, CAC, LTV)
- Vendor payouts
- Strategic planning metrics

### 5.3 Event-Style vs Snapshot-Style Data

**Event-Style (Immutable Log):**
- User behavior events (searches, views, clicks)
- Booking state transitions
- Price changes
- **Benefit:** Complete history, ML training data, audit trail
- **Storage:** Time-series database or partitioned tables (delete old data after X months)

**Snapshot-Style (Current State):**
- Vendor profiles (current contact info)
- Inventory catalog (current description, images)
- **Benefit:** Fast queries for current state
- **Storage:** Traditional RDBMS

**Hybrid Approach:**
- Maintain current state in operational DB (fast transactions)
- Stream all changes to event log (analytics, ML)
- Best of both worlds

### 5.4 How Analytics Queries Scale

**Separation of Concerns:**
```
Operational DB (PostgreSQL)
  ↓ Change Data Capture (Debezium, AWS DMS)
Data Warehouse (BigQuery/Snowflake)
  ↓ Batch processing (daily aggregations)
Analytics Dashboards (Looker, Tableau, Metabase)
```

**Query Optimization:**
- **Materialized Views:** Pre-compute expensive joins
- **Partitioning:** Partition by date (query only recent data)
- **Columnar Storage:** Warehouse optimized for analytical queries
- **Caching:** Redis cache for frequently accessed metrics (vendor dashboard)

**Read Replicas:**
- Operational DB: Read replica for reporting queries
- Prevents analytics from slowing down transactions

---

## 6. ML Roadmap (Phased Approach)

### Phase 1: ML-Ready Foundations (Months 1-6, MVP)

**Goal:** Capture data, build baselines, NO heavy models yet

**Feature Generation:**
- Item features: category, price tier, vendor rating, historical booking rate
- User features: search history, favorite categories, booking history
- Context features: season, day of week, days until event, city

**Baseline Metrics:**
- Rule-based search ranking:
  - Score = 0.4 × (vendor_rating) + 0.3 × (booking_rate) + 0.2 × (price_match) + 0.1 × (distance_from_venue)
- Simple recommendations: "Customers who booked X also booked Y" (collaborative filtering)

**Rule-Based Insights:**
- Alert vendors when utilization < 30% ("optimize your pricing")
- Alert platform when search volume > supply in category ("recruit vendors in this niche")
- Seasonality reports: "Lighting demand peaks in November-December"

**Data Infrastructure:**
- Set up event streaming to BigQuery/Snowflake
- Build daily ETL jobs for aggregations
- Create vendor analytics dashboards (no ML, just descriptive stats)

**Feedback Loops:**
- Track which search results get clicked (implicit feedback)
- Track which quotes convert to bookings (conversion signal)
- Store all booking outcomes (completed successfully, cancelled, disputed)

**No ML Models Yet:**
- Avoid premature optimization
- Ensure data quality first
- Understand domain before building models

---

### Phase 2: Predictive Analytics (Months 6-18)

**Goal:** Deploy first ML models for forecasting and optimization

**1. Demand Forecasting**
- **Model:** Time series (Prophet, ARIMA) or gradient boosting (XGBoost)
- **Input Features:**
  - Historical booking volume by category/city
  - Seasonality (month, day of week)
  - External data (wedding season calendar, holidays, weather)
- **Output:** Predicted demand for next 30/60/90 days
- **Use Case:**
  - Help vendors optimize inventory levels
  - Alert platform to recruit vendors in high-demand categories
  - Inform marketing spend allocation

**Training Data:**
- Minimum 6 months of historical bookings
- Label: number of bookings per category/city/week
- Validation: Hold-out test set (last 2 months)

**Feedback Loop:**
- Compare predictions vs actual demand weekly
- Retrain model monthly with new data

**Risk:** Data sparsity in early months → model unreliable
**Mitigation:** Start with aggregate predictions (city-level), not item-level

---

**2. Utilization Prediction**
- **Model:** Regression (predict % of inventory booked)
- **Input Features:**
  - Item characteristics (category, price, vendor rating)
  - Historical utilization rate
  - Competitive landscape (similar items in market)
- **Output:** Predicted utilization next 30 days
- **Use Case:**
  - Identify underperforming items (low predicted utilization) → suggest price drop
  - Identify over-utilized items → suggest price increase

**Training Data:**
- Label: actual utilization % (booked days / available days)
- Features: item metadata + market context

**Feedback Loop:**
- Track actual vs predicted utilization
- A/B test pricing recommendations (do vendors who follow advice see higher revenue?)

---

**3. Price Sensitivity Analysis**
- **Model:** Elasticity estimation (how demand changes with price)
- **Input:** Historical bookings at different price points
- **Output:** Demand curve per category
- **Use Case:**
  - Guide vendors on optimal pricing
  - Identify price-insensitive categories (premium decor) vs price-sensitive (basic furniture)

**Training Data:**
- Requires price variation (either natural or via A/B tests)
- Label: booking rate at given price point
- Control for confounders (seasonality, vendor quality)

**Risk:** Not enough price variation in organic data
**Mitigation:** Run controlled pricing experiments (suggest vendors test 3 price points)

---

### Phase 3: Optimization & Personalization (18+ Months)

**Goal:** Advanced ML for dynamic pricing, personalized experiences, inventory optimization

**1. Dynamic Pricing Model**
- **Model:** Reinforcement learning or contextual bandits
- **Input Features:**
  - Real-time demand signals (search volume, competitor prices)
  - Inventory levels (scarcity → higher price)
  - Customer segment (price-sensitive vs premium)
  - Time to event (last-minute bookings → premium)
- **Output:** Optimal price per item per day
- **Use Case:**
  - Maximize vendor revenue + platform GMV
  - Surge pricing for peak wedding dates

**Training Data:**
- Historical bookings with prices and outcomes
- Counterfactual reasoning: "Would booking succeed at higher price?"

**Feedback Loop:**
- A/B test dynamic prices vs static prices
- Measure impact on revenue, booking rate, customer satisfaction

**Risk:** Price fluctuations alienate customers ("price gouging")
**Mitigation:** Price caps, gradual changes, transparency ("high demand pricing")

---

**2. Personalized Search Ranking**
- **Model:** Learning-to-rank (LambdaMART, neural ranking)
- **Input Features:**
  - User preferences (inferred from search/browse history)
  - Item features (category, style, price)
  - Contextual features (event date, venue location)
- **Output:** Personalized ranking of search results
- **Use Case:**
  - Show traditional decor to users who browsed traditional items
  - Show budget options to price-sensitive users

**Training Data:**
- Label: click-through rate, booking conversion rate
- Pairwise training: item A ranked higher than B if higher CTR

**Feedback Loop:**
- Online learning: update model as users interact
- Multi-armed bandit for exploration (show some non-personalized results)

**Risk:** Filter bubble (users only see similar items)
**Mitigation:** Inject diversity (show 20% unexpected items)

---

**3. Inventory Optimization**
- **Model:** Optimization (linear programming, genetic algorithms)
- **Input:**
  - Demand forecast per category
  - Current inventory levels
  - Vendor capacity constraints
- **Output:** Recommendations for vendors (which items to add/remove)
- **Use Case:**
  - "You should add more gold-themed decor (high demand, low supply)"
  - "Consider retiring this item (low utilization)"

**Training Data:**
- Demand forecasts from Phase 2
- Vendor profitability data (ROI per item)

---

**4. Vendor Quality Scoring (Ranking Algorithm)**
- **Model:** Weighted scoring or gradient boosting
- **Input Features:**
  - Response time to booking requests
  - Completion rate (% of bookings fulfilled without issues)
  - Review ratings
  - Cancellation rate
  - Tenure on platform
- **Output:** Vendor quality score (0-100)
- **Use Case:**
  - Rank vendors in search results (high-quality vendors first)
  - Identify at-risk vendors (low score → proactive intervention)

**Training Data:**
- Label: long-term vendor success (revenue, retention)
- Features: early indicators of quality

**Feedback Loop:**
- Does prioritizing high-quality vendors improve customer satisfaction?
- Does deprioritizing low-quality vendors incentivize improvement?

**Risk:** New vendors (cold start) have no data
**Mitigation:** Default "neutral" score, fast updates as data comes in

---

### ML Infrastructure (Phase 2-3)

**Feature Store:**
- Centralized repository of ML features
- Feast, Tecton, or SageMaker Feature Store
- Serves features with low latency (<10ms) for real-time inference

**Model Training Pipeline:**
- Scheduled retraining (weekly/monthly)
- Automated hyperparameter tuning
- Model validation on hold-out test sets
- A/B testing framework for model deployment

**Model Serving:**
- REST API for inference (e.g., `/api/v1/ml/predict-demand`)
- Batch predictions for non-real-time use cases (daily demand forecast)
- Monitoring: track prediction accuracy, latency, errors

**Experiment Tracking:**
- MLflow, Weights & Biases for tracking experiments
- Version control for models and datasets

---

### ML Risk Mitigation

**1. Data Leakage:**
- **Risk:** Using future information to predict past (inflates accuracy)
- **Mitigation:** Strict train/test split by time, never use post-booking data to predict booking

**2. Feedback Loops:**
- **Risk:** Model biases reinforce (show popular items → they get more bookings → model thinks they're better)
- **Mitigation:** Exploration strategies (randomize 10-20% of rankings), causal inference

**3. Cold Start:**
- **Risk:** No data for new vendors, new items, new cities
- **Mitigation:** Content-based features (item attributes), transfer learning from other cities

**4. Model Drift:**
- **Risk:** Data distribution changes (new trends, COVID-like shocks)
- **Mitigation:** Monitor prediction accuracy over time, automated retraining, circuit breakers (fallback to rule-based)

**5. Overfitting:**
- **Risk:** Model memorizes training data, doesn't generalize
- **Mitigation:** Cross-validation, regularization, hold-out test set from different time period

---

## 7. Non-Goals (Explicitly Excluded from Phase 1)

### Features Intentionally Deferred

**1. Multi-Vendor Bookings:**
- **Excluded:** Booking items from multiple vendors in single transaction
- **Reason:** Increases complexity (coordination, split payments, delivery logistics)
- **Phase 1:** Single-vendor bookings only
- **Future:** Phase 2+ if customer demand validated

**2. Real-Time Chat:**
- **Excluded:** In-app messaging between customers and vendors
- **Reason:** Moderation burden, infrastructure complexity
- **Phase 1:** Email/phone-based communication (vendor contact visible post-booking)
- **Future:** Consider if response time becomes bottleneck

**3. Advanced Logistics:**
- **Excluded:** Delivery tracking, route optimization, driver assignment
- **Reason:** Vendors self-manage delivery in Phase 1
- **Phase 1:** Vendor responsible for delivery (trust-based)
- **Future:** Platform-managed logistics if scale justifies

**4. Multi-Currency/Multi-Language:**
- **Excluded:** Full internationalization
- **Reason:** Focus on India first (single language: English/Hindi, currency: INR)
- **Phase 1:** India-only
- **Future:** Expand to other markets with validated product-market fit

**5. White-Label Vendor Storefronts:**
- **Excluded:** Custom domains for vendors (e.g., vendor.weddingrentals.com)
- **Reason:** Adds complexity, dilutes marketplace brand
- **Phase 1:** All vendors on platform domain
- **Future:** Premium feature for enterprise tier

**6. Advanced Dynamic Pricing (Surge Pricing):**
- **Excluded:** Real-time algorithmic pricing
- **Reason:** Requires ML models, data, vendor trust
- **Phase 1:** Static pricing with seasonal rules
- **Future:** Phase 3 ML roadmap

**7. Social Features:**
- **Excluded:** User-generated content (photos from weddings), social sharing, wishlists
- **Reason:** Nice-to-have, not core to MVP validation
- **Phase 1:** Basic reviews only
- **Future:** If community engagement becomes differentiator

**8. Vendor Analytics Mobile App:**
- **Excluded:** Native iOS/Android apps for vendors
- **Reason:** Resource-intensive, mobile web sufficient initially
- **Phase 1:** Responsive web dashboard
- **Future:** If mobile vendor usage > 60%

### ML Ideas Deferred

**1. Image Recognition:**
- Auto-tagging items from photos (detect "gold", "floral", "traditional")
- **Why Deferred:** Requires labeled training data, complex CV models
- **When:** Phase 3+ if manual tagging becomes bottleneck

**2. Natural Language Search:**
- Semantic search ("decor for outdoor garden wedding") vs keyword matching
- **Why Deferred:** Requires embeddings, vector search infrastructure
- **When:** Phase 2-3 if search satisfaction < 70%

**3. Predictive Overbooking:**
- Intentionally overbook inventory (airline-style) based on cancellation predictions
- **Why Deferred:** High risk (customer trust), requires accurate cancellation models
- **When:** Never (too risky for weddings)

**4. Customer Lifetime Value Prediction:**
- Predict which customers will book repeatedly
- **Why Deferred:** Weddings are typically one-time events (low repeat rate)
- **When:** Only if expanding to event planners (repeat customers)

### UX/Product Complexity Avoided

**1. Auctions/Bidding:**
- Vendors bid for customer requests (reverse auction)
- **Why Avoided:** Adds friction, delays booking confirmation
- **Approach:** Fixed pricing + quote system

**2. Subscription Model for Customers:**
- Monthly fee for unlimited bookings
- **Why Avoided:** Weddings are one-time, subscriptions don't fit
- **Approach:** Per-booking fees only

**3. Gamification:**
- Vendor badges, leaderboards, achievement systems
- **Why Avoided:** Distracts from core value (quality inventory, reliable bookings)
- **Approach:** Simple reputation system (ratings + performance metrics)

**4. Customization/Build-Your-Own:**
- Design custom decor packages via UI builder
- **Why Avoided:** Requires vendor design capabilities, complex pricing
- **Approach:** Customers describe needs, vendors quote manually (Phase 1)

---

## 8. Risks & Mitigations

### 8.1 Data Sparsity Issues

**Risk:** Early months = low transaction volume → unreliable analytics/ML

**Impact:**
- Demand forecasts have wide confidence intervals
- Vendor performance metrics noisy (10 bookings not statistically significant)
- Recommendation engines suffer from cold start

**Mitigations:**
1. **Aggregate Early:** City-level trends before item-level
2. **External Data:** Wedding seasonality data, Google Trends for validation
3. **Bayesian Priors:** Use industry benchmarks as priors (wedding season = Oct-Feb in India)
4. **Transparency:** Show confidence intervals in dashboards ("low confidence" warnings)
5. **Rule-Based Fallbacks:** When data insufficient, use expert rules

**Monitoring:**
- Track data coverage: % of items with >10 bookings, >100 views
- Alert when making predictions on sparse data

---

### 8.2 Cold-Start Vendors

**Risk:** New vendors have no reviews, no booking history → customers don't trust → no bookings → vicious cycle

**Impact:**
- Vendor churn (onboard, get no bookings, leave)
- Marketplace inventory gaps

**Mitigations:**
1. **Curated Onboarding:** Manually vet first vendors, promote high-quality profiles
2. **Neutral Ranking:** New vendors start at median position (not bottom)
3. **Fast Feedback:** After first 3 bookings, update ranking (don't wait for 20+)
4. **Highlight "New" Status:** Badge for new vendors, some customers prefer trying new vendors
5. **Platform Guarantees:** "Platform-verified vendor" badge after vetting
6. **Subsidize First Bookings:** Platform co-marketing for new vendors (featured listings for first month)

**Monitoring:**
- Track time-to-first-booking for new vendors
- Target: <14 days, alert if >30 days

---

### 8.3 Seasonality Spikes

**Risk:** Wedding season (Oct-Feb in India) → demand spike → inventory shortages → poor customer experience

**Impact:**
- Double-booking attempts increase
- Vendor response times slow (overwhelmed)
- Customer acquisition cost wasted (traffic but no inventory)

**Mitigations:**
1. **Demand Forecasting:** Predict spikes, recruit vendors proactively (3 months ahead)
2. **Dynamic Inventory Buffers:** Reduce displayed availability in peak season (show 80% of actual capacity)
3. **Vendor Incentives:** Bonus commission for vendors who add inventory pre-season
4. **Customer Expectation Setting:** Show "high demand" warnings, suggest booking early
5. **Waitlists:** If item unavailable, capture lead for similar items
6. **Flexible Date Search:** Suggest nearby dates if preferred date booked out

**Monitoring:**
- Track inventory utilization rates (alert when >70% city-wide)
- Track search-to-booking conversion (drops during shortages)

---

### 8.4 Vendor Data Quality

**Risk:** Vendors enter incomplete/inaccurate data → poor customer experience → platform blame

**Examples:**
- Missing images → customers don't book
- Wrong availability → double-bookings
- Inflated item descriptions → negative reviews
- Inconsistent categorization → search doesn't work

**Mitigations:**
1. **Guided Onboarding:** Step-by-step wizard with validation rules
   - Require minimum 3 photos per item
   - Enforce category selection (no free text)
   - Validation: price within reasonable range for category
2. **Quality Scores:** Grade vendor listings (0-100), show score in vendor dashboard
   - Incomplete listings → lower search ranking
3. **Auto-Populate:** Suggest item attributes based on title/category (ML in Phase 2)
4. **Vendor Education:** Best practices guide, examples of high-quality listings
5. **Periodic Audits:** Random sample review, contact vendors with low scores
6. **Customer Feedback Loop:** "Report inaccurate listing" button → platform review

**Monitoring:**
- % of items with complete data (target: >90%)
- Correlation between listing quality and booking rate

---

### 8.5 Overfitting ML Early

**Risk:** Build complex ML models prematurely → waste resources, unreliable predictions

**Symptoms:**
- Models work great on training data, fail in production
- Obsessing over 0.01% accuracy gains
- Building features that won't generalize

**Mitigations:**
1. **Phase 1: No ML** - Prove business model first, collect data
2. **Start Simple:** Linear models, decision trees before deep learning
3. **Business Metric Focus:** Optimize for revenue/conversion, not accuracy scores
4. **Holdout Sets:** Strict temporal validation (train on Jan-Mar, test on Apr)
5. **Avoid Early Optimization:** Don't build real-time inference until batch predictions validated
6. **Fail Fast:** If model doesn't beat baseline in 2 weeks, abandon and try simpler approach

**Monitoring:**
- Track model vs baseline (rule-based) performance
- Require 10%+ improvement to justify deployment

---

## 9. MVP vs Scale-Up Boundary

### 9.1 MVP Success Criteria (Exit Criteria for Phase 1)

**Business Validation:**
- [ ] 30+ active vendors (at least 1 booking in past 30 days)
- [ ] 100+ bookings completed successfully
- [ ] 60%+ booking fulfillment rate (bookings completed without cancellation/disputes)
- [ ] 50%+ vendor satisfaction (would recommend platform to other vendors)
- [ ] 40%+ customer satisfaction (NPS >0)
- [ ] Unit economics positive (LTV > CAC)

**Product Validation:**
- [ ] <5% double-booking rate (conflicts per bookings attempted)
- [ ] 70%+ search satisfaction ("I found what I needed")
- [ ] <24hr vendor response time (median)
- [ ] Payment system working (0 failed transactions due to platform errors)

**Technical Validation:**
- [ ] 99.5%+ uptime
- [ ] <2s page load time (p95)
- [ ] Database can handle 1000 bookings/month (stress tested)

**When to Scale Up:**
- All above metrics achieved in **single city** (e.g., Mumbai)
- Operational playbook documented (vendor onboarding, customer support, dispute resolution)
- Team confident in expanding to second city

---

### 9.2 Scale-Up Phase Triggers

**Expand to Multi-City When:**
- Single city profitable (GMV × take_rate > operating costs)
- Vendor NPS >30 (would recommend)
- Operational processes documented and repeatable
- Hiring plan ready (city managers for new locations)

**Invest in ML When:**
- 6+ months of data (minimum for reliable models)
- Business metrics plateauing (can't improve with product alone)
- Data infrastructure stable (clean pipelines, <1% data quality issues)

**Build Microservices When:**
- Monolith deployment taking >30 min
- Teams stepping on each other (code conflicts)
- Scaling bottlenecks (can't scale booking service independently)

**Switch to Elasticsearch When:**
- Algolia costs >$5k/month
- Need custom ranking algorithms
- Have ML team to maintain search relevance

---

### 9.3 What Stays the Same, What Changes

**Keep in MVP → Scale:**
- Event sourcing for bookings (foundational decision)
- Pool model multi-tenancy (works for 80% of vendors)
- Stripe Connect payments (industry standard)
- Single-vendor bookings (avoid complexity)

**Change in Scale-Up:**
- Monolith → Microservices (booking, inventory, search, payments as separate services)
- Algolia → Elasticsearch (cost optimization, customization)
- Single DB → Read replicas + data warehouse
- Manual vendor vetting → Automated verification workflows
- Rule-based ranking → ML-based ranking

---

## 10. Cultural Considerations: The "Reuse Stigma" Challenge

### 10.1 The Problem

In Indian wedding culture, there is a **significant stigma around reusing items from others' weddings**, particularly for personal/intimate items. Weddings are once-in-a-lifetime events, and the perception of wearing "someone else's wedding dress" or using "second-hand" decor can be emotionally charged and culturally sensitive.

**Risk if not addressed:**
- Customer hesitation/rejection of the entire rental concept
- Negative word-of-mouth ("they're selling used wedding items")
- Platform positioned as "budget/cheap" option (aspirational couples avoid)
- Religious/cultural objections (purity concerns for certain items)

### 10.2 Category-Based Strategy: What to Rent vs Avoid

**Tier 1: HIGH Acceptance - Focus MVP Here** ✓
Categories with minimal stigma, already commonly rented in India:

1. **Furniture** (Tables, Chairs, Sofas, Stages)
   - Already normalized rental category
   - Purely functional, no emotional attachment
   - High cost of ownership (storage, maintenance)
   - **Positioning:** "Premium event furniture rental"

2. **Decor Elements** (Centerpieces, Backdrops, Drapes, Floral Arrangements)
   - Event-specific, not personal
   - High variety desired (different for each function)
   - Expensive to buy and use once
   - **Positioning:** "Curated wedding decor collections"

3. **Lighting** (Chandeliers, Uplighting, String Lights, LED panels)
   - Technical equipment, zero emotional stigma
   - Requires expertise to set up
   - **Positioning:** "Professional event lighting"

4. **Props & Installations** (Photo booth setups, Entrance gates, Mandap structures)
   - Temporary installations
   - Already standard rental practice
   - **Positioning:** "Instagram-worthy wedding setups"

**Tier 2: MODERATE Acceptance - Phase 2 with Careful Positioning**

5. **Jewelry** (Costume jewelry, statement pieces for pre-wedding shoots)
   - Growing acceptance for non-bridal functions (mehendi, sangeet)
   - Position as "fashion jewelry," NOT "bridal heirlooms"
   - **Risk Mitigation:** Emphasize professional cleaning, quality certificates

6. **Accessories** (Bags, Clutches, Shoes for bridesmaids)
   - Acceptable for non-bride participants
   - **Positioning:** "Designer accessories for your squad"

**Tier 3: HIGH Stigma - AVOID in MVP (Defer indefinitely or exclude)**

7. **Wedding Outfits** (Bride's lehenga, Groom's sherwani) ⚠️
   - **HIGHEST stigma** - deeply personal, emotional significance
   - Religious considerations (purity for wedding attire)
   - "Someone else wore this on THEIR special day"
   - **Recommendation:** EXCLUDE from platform entirely OR only offer for pre-wedding functions (NOT main ceremony)
   - **Alternative Model:** Partner with designers for "made-to-order rentals" (new pieces, rented once, then repurposed)

8. **Intimate/Personal Items** (Dupattas worn by bride, Mangalsutra, Sacred items)
   - Strong cultural/religious objections
   - **Recommendation:** NEVER offer rentals for these

### 10.3 Product Positioning & Messaging Strategy

**Language Matters - NEVER Use:**
- ❌ "Second-hand"
- ❌ "Used"
- ❌ "Pre-owned"
- ❌ "Rental" (in some contexts)

**Instead, Use:**
- ✓ "Curated wedding collections"
- ✓ "Premium event solutions"
- ✓ "Designer decor experiences"
- ✓ "Sustainable luxury" (reframe as eco-conscious, not cheap)
- ✓ "Access over ownership" (modern, minimalist positioning)

**Brand Positioning:**
- **NOT:** "Save money by renting used wedding stuff"
- **YES:** "Create your dream wedding with premium, curated collections without the burden of ownership"

**Sustainability Angle (for Metro audiences like Bangalore):**
- Position rentals as **eco-conscious** choice
- "Reduce waste, celebrate sustainably"
- Appeal to modern, educated couples who value environmental impact
- Success examples: Rent the Runway (US), Flyrobe (India for fashion)

### 10.4 Trust & Hygiene Signals (Critical for Acceptance)

**Transparency in Item Lifecycle:**
1. **"Brand New" vs "Pristine Collection"** labeling
   - Some items marked "New for this season" (premium pricing)
   - Others "Professionally cleaned & inspected" (standard pricing)
   - NEVER hide that items are reused

2. **Hygiene & Quality Standards:**
   - Mandatory professional cleaning certificates (visible in listing)
   - "Sanitized & inspected" badge
   - Photos of cleaning process (build trust)
   - Quality score: "Condition: Like New / Excellent / Good"

3. **Vendor Transparency:**
   - Show vendor's cleaning/maintenance process
   - Reviews mentioning item condition
   - "This vendor maintains 98% 'Like New' condition ratings"

**Social Proof:**
- Showcase real weddings using rented decor (with permission)
- Testimonials emphasizing quality, not just price savings
- Influencer partnerships (wedding bloggers, photographers)

### 10.5 Data Model Implications

**Item Metadata - Add Fields:**
```sql
ALTER TABLE inventory_items ADD COLUMN item_condition VARCHAR(20);
-- 'brand_new', 'like_new', 'excellent', 'good'

ALTER TABLE inventory_items ADD COLUMN times_rented INT DEFAULT 0;
-- Track usage, customer can see rental history

ALTER TABLE inventory_items ADD COLUMN last_cleaned_at TIMESTAMPTZ;
-- Hygiene transparency

ALTER TABLE inventory_items ADD COLUMN cleaning_certificate_url VARCHAR;
-- Link to cleaning/inspection docs

ALTER TABLE inventory_items ADD COLUMN is_intimate_item BOOLEAN DEFAULT FALSE;
-- Flag items that should NEVER be rented (future safeguard)
```

**Category-Specific Attributes:**
- Add `category_stigma_level` to categories table (high/medium/low)
- Filter out high-stigma categories in customer search by default
- Track customer sentiment on rental acceptance per category (ML feature)

### 10.6 Marketing & Customer Education

**Pre-Launch Content Strategy:**
1. **Blog/SEO Content:**
   - "Why Top Wedding Planners Choose Rental Decor"
   - "The Rise of Sustainable Luxury Weddings in Bangalore"
   - "How to Create a 50L Wedding Look on a 20L Budget" (subtle rental messaging)

2. **Vendor Partnerships:**
   - Partner with wedding planners who already recommend rentals
   - Get testimonials from respected planners ("I only work with [Platform] for decor")

3. **Instagram-First Strategy:**
   - Showcase stunning wedding setups (NOT the couple, the decor)
   - Before/after transformations
   - "This entire setup cost ₹2L, not ₹10L" (price transparency)

**Customer Onboarding:**
- FAQ addressing stigma directly (subtle, not defensive)
  - "Are rental items clean?" → Show cleaning process
  - "Will items look worn out?" → Condition ratings, quality guarantees
  - "Is renting looked down upon?" → Reframe as modern, sustainable choice

### 10.7 Monitoring & Iteration

**Track Sentiment Metrics:**
- Category-wise conversion rates (which categories have high browse, low book → stigma signal)
- Customer feedback: "Why didn't you book?" survey
- Track which messaging resonates (A/B test "rental" vs "curated collections")

**Early Warning Signals:**
- High cart abandonment on specific categories (e.g., jewelry) → stigma issue
- Negative reviews mentioning "used" or "unclean" → quality control failure
- Low repeat bookings → trust issue

**Pivot Indicators:**
- If outfit rentals have <5% conversion despite traffic → exclude category
- If customers consistently filter for "brand new only" → adjust inventory strategy
- If organic word-of-mouth is negative → rebrand entirely

### 10.8 Competitive Differentiation on Stigma

**How Competitors Handle This:**
- **Flyrobe (India):** Fashion rental, positioned as "designer access," not "used clothes"
- **Wedding vendors:** Many avoid "rental" terminology entirely, use "event solutions"

**Our Approach:**
- **Premium positioning** from day one (not discount brand)
- **Quality over quantity** (curated vendors, strict condition standards)
- **Transparency builds trust** (show cleaning process, condition ratings)
- **Start with low-stigma categories** (furniture, decor, lighting)
- **Never compromise on hygiene** (visible quality standards)

### 10.9 Recommended MVP Category Focus (Bangalore Launch)

**Phase 1 MVP - Launch with ONLY Low-Stigma Categories:**
1. Furniture (tables, chairs, stages)
2. Decor (centerpieces, backdrops, drapes)
3. Lighting (uplighting, chandeliers, string lights)
4. Props (photo booths, entrance gates)

**Explicitly EXCLUDE from MVP:**
- Wedding outfits (bride/groom attire)
- Intimate accessories (dupattas, jewelry for main ceremony)
- Sacred/religious items

**Phase 2 Expansion (Post-6 months, if data supports):**
- Jewelry for pre-wedding functions only
- Groomsmen/bridesmaid outfits (NOT bride/groom)
- Accessories (bags, shoes for non-bride participants)

**NEVER Offer (High Risk/Negative ROI):**
- Bridal lehenga/saree for main wedding ceremony
- Mangalsutra, sacred threads, religious items
- Undergarments, intimate wear

---

## 11. Key Decisions Made & Remaining Questions

### Decisions Finalized ✓

1. **Target Customer:** Couples planning their own weddings (B2C direct)
2. **Launch City:** Bangalore (tech-savvy, manageable market size)
3. **Revenue Model:** Commission-only, 10-20% per booking
4. **Vendor Mobile:** Native mobile app required (React Native for MVP)

### Remaining Strategic Questions

**Vendor Acquisition:**
1. How will we recruit the first 30 Bangalore vendors?
   - Manual outreach to established rental companies?
   - Partnerships with wedding venues for vendor referrals?
   - Paid ads targeting "wedding rental business Bangalore"?

2. What's the vendor incentive to join?
   - Zero commission for first 3 months?
   - Featured placement in search results?
   - Exclusive category territory (first mover advantage)?

**Operations & Logistics:**
3. Delivery model for MVP?
   - **Option A:** Vendor self-manages delivery (hands-off, lower complexity)
   - **Option B:** Platform provides delivery partners (better customer experience, higher complexity)
   - **Recommendation:** Option A for MVP, add managed logistics in Phase 2

4. Commission rate differentiation?
   - Flat 15% across all categories (simpler)?
   - Tiered by category: 20% for high-margin (decor), 10% for low-margin (furniture)?
   - **Recommendation:** Start flat 15%, optimize after 6 months of data

**Technical & Build/Buy:**
5. Payment provider priority?
   - **Razorpay** (India-optimized, UPI, lower fees, local support)
   - **Stripe Connect** (global standard, better docs, easier international expansion)
   - **Recommendation:** Razorpay for India launch, maintain Stripe integration for future global markets

6. Analytics platform?
   - Build custom event tracking + BigQuery warehouse (full control, lower long-term cost)
   - Use Mixpanel/Amplitude (faster MVP, higher cost at scale)
   - **Recommendation:** Custom event tracking to BigQuery (data ownership critical for ML future)

**Unit Economics Assumptions (Need Validation):**
7. Expected average order value (AOV)?
   - Conservative: ₹15,000 (basic decor + furniture rental)
   - Optimistic: ₹40,000 (full wedding package)
   - **Action:** Validate with vendor interviews in Bangalore

8. Expected customer acquisition cost (CAC)?
   - Organic (SEO, content): ₹500-1,000 per booking
   - Paid (Google/Instagram ads): ₹2,000-4,000 per booking
   - **Action:** Budget ₹2,000 CAC assumption, optimize post-launch

9. Expected vendor acquisition cost?
   - Manual outreach: ₹5,000 per vendor (time cost)
   - Paid ads: ₹10,000-20,000 per vendor
   - **Action:** Start with manual outreach to control quality, measure activation rate

**Competitive Positioning:**
10. How do we differentiate from WedMeGood, WeddingWire India?
    - **WedMeGood:** Primarily vendor directory (lead generation), not transactional marketplace
    - **WeddingWire:** Similar model, but weaker in rentals (stronger in venues/photographers)
    - **Our Differentiation:**
      - Transactional marketplace (book + pay on platform, not just leads)
      - Superior search/discovery (Algolia, ML-powered ranking in Phase 2)
      - Vendor mobile-first (better vendor experience = faster response times)
      - Data-driven pricing insights for vendors (help them optimize revenue)

---

## Implementation Phases Summary

### Phase 1: MVP (Months 0-6)
**Focus:** Validate marketplace fit in Bangalore

**Deliverables:**
- Customer web + mobile web (PWA) - search, booking flow
- Vendor mobile app (React Native - iOS + Android)
- Admin panel (vendor approval, analytics)
- PostgreSQL database (pool model multi-tenancy)
- Algolia search integration
- Razorpay/Stripe payments (India-optimized)
- Basic analytics (descriptive stats, no ML)

**Team:** 3-4 engineers (2 full-stack, 1 mobile specialist, 1 backend), 1 designer, 1 product manager

**Tech Stack:**
- Backend: Node.js/Express or Python/FastAPI
- Customer Frontend: React/Next.js (SSR for SEO)
- Vendor Mobile: React Native (iOS + Android)
- Database: PostgreSQL 15+
- Cache: Redis
- Search: Algolia (managed)
- Payments: Razorpay (India-first) with Stripe as backup
- Hosting: AWS Mumbai region (low latency for Bangalore)

---

### Phase 2: Growth (Months 6-18)
**Focus:** Expand to 2-3 cities, optimize operations

**Deliverables:**
- Multi-city expansion
- Advanced search (filters, facets)
- Vendor mobile app (React Native)
- Data warehouse (BigQuery/Snowflake)
- First ML models (demand forecasting, utilization prediction)
- CQRS for bookings (separate read/write paths)
- Microservices (split monolith)

**Team:** 5-7 engineers (backend, frontend, mobile, data), 1-2 designers, 1 PM, 1 data analyst

---

### Phase 3: Scale (Months 18+)
**Focus:** National scale, ML optimization, operational excellence

**Deliverables:**
- 10+ cities
- Advanced ML (dynamic pricing, personalized ranking)
- Multi-region infrastructure
- Event-driven architecture
- Feature flags, A/B testing framework
- Real-time analytics dashboards
- Vendor self-serve onboarding (automated verification)

**Team:** 10+ engineers (specialized roles), data science team, DevOps, QA

---

## Critical Files to Be Created (Implementation Phase)

### Backend Services
- `services/booking/booking-service.ts` - Booking workflow, event sourcing
- `services/inventory/inventory-service.ts` - Item CRUD, availability management
- `services/vendor/vendor-service.ts` - Vendor profiles, verification
- `services/search/search-service.ts` - Algolia integration, ranking
- `services/payment/payment-service.ts` - Stripe Connect, split payments
- `services/analytics/analytics-service.ts` - Event tracking, aggregations

### Database Schemas
- `db/migrations/001_create_vendors.sql`
- `db/migrations/002_create_inventory_items.sql`
- `db/migrations/003_create_bookings.sql`
- `db/migrations/004_create_booking_events.sql`
- `db/migrations/005_create_item_availability.sql`
- `db/migrations/006_create_analytics_tables.sql`

### API Routes
- `api/v1/customer/search.ts`
- `api/v1/customer/bookings.ts`
- `api/v1/vendor/inventory.ts`
- `api/v1/vendor/bookings.ts`
- `api/v1/vendor/analytics.ts`

### Data Pipelines
- `pipelines/event-ingestion.ts` - Stream events to warehouse
- `pipelines/daily-aggregations.ts` - Compute analytics tables
- `pipelines/vendor-performance.ts` - Calculate vendor scores

### Frontend
- `app/customer/search/page.tsx` - Search UI
- `app/customer/bookings/page.tsx` - Booking management
- `app/vendor/dashboard/page.tsx` - Vendor analytics
- `app/vendor/inventory/page.tsx` - Inventory management

---

## Verification Plan (Post-Implementation)

### End-to-End Testing Scenarios

**1. Customer Booking Flow:**
- Search for "gold mandap" in Bangalore for Feb 14, 2026
- Filter by price <50k
- View item details, check availability
- Request quote
- Confirm booking
- Make payment
- **Verify:** Booking created, vendor notified, payment split correctly, availability updated

**2. Vendor Workflow:**
- Create new inventory item (upload photos, set pricing)
- Set availability calendar
- Receive booking request
- Confirm booking
- Mark booking as fulfilled
- **Verify:** Item searchable, bookings appear in dashboard, payout scheduled

**3. Double-Booking Prevention:**
- Item A has 1 unit available for Feb 14
- Customer 1 books item A for Feb 14
- Customer 2 attempts to book item A for Feb 14 (concurrent request)
- **Verify:** Second booking rejected with "item unavailable" error

**4. Analytics Pipeline:**
- Customer views item → event logged
- Customer books item → event logged
- Wait for daily aggregation job
- **Verify:** Vendor dashboard shows updated view count, booking count, revenue

**5. Search Quality:**
- Search "traditional decor" → verify relevant items ranked highly
- Search "cheap chairs" → verify price-sorted results
- Search with typo "mndap" → verify autocorrect suggests "mandap"

### Load Testing
- Simulate 100 concurrent bookings → verify no double-bookings
- Simulate 1000 searches/minute → verify <2s response time
- Test database connection pool under load

### Data Quality Checks
- Verify no NULL vendor_ids in inventory_items
- Verify booking events chronologically ordered
- Verify payment splits add up to total amount
- Verify availability quantity never goes negative

---

## Conclusion

This architecture is designed for a **data-first, ML-ready wedding rentals marketplace** launching in **Bangalore** for **couples planning weddings**, with **commission-based revenue** and **mobile-first vendor experience**. The plan scales from MVP to national platform with these key principles:

1. **Incremental complexity** - Start simple (CRUD), add sophistication (event sourcing, ML) when justified
2. **Analytics from day one** - Capture all events, even if not analyzing immediately
3. **Boring reliability** - Proven patterns (PostgreSQL, Razorpay) over novel architectures
4. **Vendor-centric design** - Vendors are the constrained resource, native mobile app critical
5. **Bangalore validation** - Prove unit economics in tech-savvy market before expanding to Mumbai/Delhi

The plan balances **pragmatic MVP** (can ship in 3-4 months) with **long-term scalability** (supports ML, multi-region, microservices when needed). It avoids common pitfalls (premature ML, over-engineering, scaling too fast) while building foundations that won't require rewrites.

---

## Quick Reference: Critical Architecture Decisions

### Data Architecture
- **Multi-Tenancy:** Pool model (shared schema, vendor_id discriminator)
- **Event Sourcing:** Bookings + payments only (not full CQRS initially)
- **Conflict Prevention:** Database-level unique constraints on (item_id, date)
- **Analytics:** Custom event streaming → BigQuery (data ownership for ML)

### Technology Stack (MVP)
- **Backend:** Node.js/Express or Python/FastAPI
- **Customer:** React/Next.js (web + PWA)
- **Vendor:** React Native (iOS + Android native app)
- **Database:** PostgreSQL 15+ (AWS RDS Mumbai region)
- **Cache:** Redis (session data, hot availability queries)
- **Search:** Algolia (managed, fast MVP, migrate to Elasticsearch in Phase 2)
- **Payments:** Razorpay (India UPI support) + Stripe (global backup)

### API Design
- **Versioning:** URL-based (`/api/v1/...`)
- **Auth:** JWT with vendor_id claims
- **Rate Limiting:** Per-vendor throttling
- **GraphQL:** Deferred to Phase 2 (REST sufficient for MVP)

### ML Roadmap Summary
- **Phase 1 (Months 0-6):** Data capture only, rule-based ranking, NO models
- **Phase 2 (Months 6-18):** Demand forecasting, utilization prediction, price sensitivity
- **Phase 3 (Months 18+):** Dynamic pricing, personalized search, inventory optimization

### Success Metrics (MVP Exit Criteria)
- 30+ active Bangalore vendors
- 100+ completed bookings
- 60%+ booking fulfillment rate
- <5% double-booking rate
- Unit economics: LTV > CAC
- 99.5%+ uptime

**Next Steps:**
1. Validate unit economics assumptions (AOV, CAC, vendor acquisition cost) with Bangalore market research
2. Recruit first 10 vendors manually to inform platform design
3. Proceed to implementation with clear phase boundaries
