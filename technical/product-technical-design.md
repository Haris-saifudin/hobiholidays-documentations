# Product Domain — Technical Data Model & Architecture

> **Overview**
> Single source of truth for the Product Domain PostgreSQL schema. This document covers the complete DDL for all product tables, architectural principles, and per-table ERDs with sample data.
>
> _Engineered for High Scalability, Data Integrity, and optimized for a NestJS + PostgreSQL stack._

**Document Map:**

| Document                                                       | Responsibility                                                               |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **This file**                                                  | Full DDL schema, per-table ERDs, sample data, architecture principles        |
| [Product Hierarchy](./product-hierarchy-technical-design.md)   | 3-level hierarchy mental model, full-domain ERD, hierarchy sample data (GWE) |
| [Product Media](./product-media-technical-design.md)           | Media asset repository, polymorphic usages, presigned uploads, CDN delivery  |
| [Search & Filter](./product-search-filter-technical-design.md) | Search API contract, SQL query, indexing strategy                            |
| [Area Domain](./area-technical-design.md)                      | 4-tier geography tree (Continent → Sub Continent → Country → POI), pure relational model |
| [SEO Architecture](./seo-technical-design.md)                  | SEO metadata, Schema.org rich snippets, Next.js dynamic metadata             |
| [API Contracts](../contracts/README.md)                         | Complete REST API contracts, split sub-resources, request/response DTOs      |
| [Backend Guide](../backend/product-backend-guide.md)            | NestJS ProductModule, services, transactions, and split endpoints             |
| [Frontend Guide](../frontend/product-frontend-guide.md)          | Next.js 15 PDP rendering, tabbed UI loading, and brochure download           |

---

## 🏗️ Architecture & Engineering Principles

### 1. Client-Optimized Data Aggregation

Backend must serve aggregated JSON payloads — never raw rows. Avoid N+1 queries by joining variants, trips, pricing, and media in a **single database round-trip** using TypeORM query builders or raw SQL with `json_agg`.

### 2. Schema Version Control

All DDL scripts are the source of truth for ORM migrations. Every schema change must be committed as a versioned migration file and run through CI/CD pipelines — never applied manually in production.

### 3. Polymorphic Relationships

Media usages and supplementary content use `(target_type, target_id)` to target multiple entity types from one table. Rules:

- **DB-level:** composite B-Tree index on `(target_type, target_id)` is mandatory for read performance.
- **App-level:** NestJS service layer is responsible for cascading deletes to polymorphic child rows inside a **database transaction** — no database FK can enforce this.
- **Valid `target_type` values:** `PRODUCT` · `VARIANT` · `TRIP` · `ITINERARY_ITEM`

### 4. Catalog Quota & Nominal Availability (Decoupled Concurrency Locking)

- **Read-Only Availability Representation:** `product_trips.max_quota` and `min_quota` serve as nominal departure capacity limits surfaced to travelers during catalog discovery and on PDP schedules:
  $$\text{availableSeats} = \max(0, \text{max\_quota} - \text{booked\_seats})$$
- **Downstream Delegation:** Transactional pessimistic concurrency locking (`SELECT ... FOR UPDATE`), mutex quota deductions, and lock TTL mechanisms are strictly decoupled from the catalog domain and delegated downstream to Phase 3 (Booking & Checkout Domain). Catalog endpoints provide instantaneous O(1) read-only metric evaluation.

### 5. Safe & Idempotent Catalog Lifecycle (No Hard Cascade Delete Required)

- **Non-Destructive Synchronization:** Catalog synchronization from ATW (All Tours Website) must be non-destructive and idempotent.
- **State Machine & Soft Deletion:** Entities are governed by `listing_status` (`'ACTIVE'`, `'INACTIVE'`, `'ARCHIVED'`) and soft-delete timestamps (`deleted_at TIMESTAMP NULL`).
- **Cascade Independence:** Archiving or deactivating a master product (`listing_status = 'ARCHIVED'`) automatically excludes child variants and trips from public search feeds without requiring destructive database drops (`DELETE CASCADE`).

### 6. Decoupled PostGIS & Pure Relational Geography

The platform eliminates all PostGIS extensions, spatial geometry types (`GEOMETRY`), GiST indexes, and spatial query operators (`ST_Contains`, `ST_Within`). Geographic hierarchy traversal relies on standard B-Tree indexing on `(parent_id, area_type_id, slug)` and trigram matching. Only `"uuid-ossp"` and `"pg_trgm"` are retained.

### 7. Precision Economics

All price columns use `DECIMAL(15,2)`. Never use `FLOAT` or `DOUBLE` for monetary values. Use `decimal.js` or `big.js` in NestJS for all arithmetic before returning to clients.

### 8. Media Lifecycle & Validation (Images & Videos)

`product_media` serves as the centralized repository for marketing visual assets: images and videos.

- **Visual Asset MIME & Byte Validation:** Image and video uploads require strict backend validation for allowed MIME types (`image/jpeg`, `image/png`, `image/webp`, `video/mp4`) along with file header verification in the NestJS upload interceptor. Maximum file size is strictly capped (e.g., 25 MB).
- **Single Cover Image Guarantee:** A partial unique index (`uq_media_usages_single_cover`) on `product_media_usages` guarantees at the database level that an entity target can have at most **one** active `COVER` image.
- **Client Cache Optimization:** Media records store `file_name` and `file_size_bytes` so UI clients can display responsive image sets. Binary streaming headers use immutable 1-year browser caching (`Cache-Control: public, max-age=31536000, immutable`).
- **Tour Itinerary PDF Brochure (External ATW Generation):** Official tour brochure PDFs are generated and hosted externally by **ATW**. Hobiholidays does not process, upload, or store PDF binaries in `product_media`. Instead, `products.itinerary_pdf_url` (default) and `product_variants.itinerary_pdf_url` (variant edition override) store the external ATW brochure URL directly for instant O(1) reads resolved via `COALESCE(v.itinerary_pdf_url, p.itinerary_pdf_url)`.

### 9. Audit Timestamps & State Traceability (`created_at`, `updated_at`, `deleted_at`)

Every domain table strictly maintains timestamp tracking for auditability, cache invalidation, and data synchronization:

- **`created_at` & `updated_at`:** Every table enforces `TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP`. A database trigger function (`set_updated_at_timestamp()`) automatically updates `updated_at` on row mutation to prevent stale data even during direct SQL operations.
- **Soft Deletes (`deleted_at`):** High-value catalog entities (`products`, `product_variants`) use nullable `deleted_at` timestamps instead of physical deletion. Partial indexes explicitly exclude soft-deleted rows (`WHERE deleted_at IS NULL`) to maintain query performance.

### 10. Catalog Lifecycle State Machine & Classification Enums

All entity lifecycles, audience classifications, and category tiers are enforced strictly via database-level `CHECK` constraints:

#### A. Listing Status (`listing_status`) — `products` & `product_variants`

Governs catalog visibility, publication readiness, and public bookability across the tour lifecycle:

- **`DRAFT`:** Initial draft state; visible only to the author/operator; not indexed in search or bookable.
- **`PENDING_REVIEW`:** Submitted by tour creator / merchant for editorial and compliance verification; awaiting administrator approval.
- **`ACTIVE`:** Verified, approved, and live on the storefront; indexed in search queries; bookable if valid departure trips exist.
- **`INACTIVE`:** Temporarily paused / hidden from catalog (e.g., seasonal hiatus, content revamp); not bookable or searchable publicly.
- **`SUSPENDED`:** Administratively frozen / locked due to safety, regulatory, policy violation, or merchant suspension.
- **`ARCHIVED`:** Permanently retired catalog item; preserved strictly for historical booking references and financial audit trails.

```mermaid
stateDiagram-v2
    [*] --> DRAFT
    DRAFT --> PENDING_REVIEW : Submit for Review
    PENDING_REVIEW --> DRAFT : Revisions Required
    PENDING_REVIEW --> ACTIVE : Approve & Publish
    ACTIVE --> INACTIVE : Pause / Seasonal Hiatus
    INACTIVE --> ACTIVE : Reactivate
    ACTIVE --> SUSPENDED : Administrative Lock
    SUSPENDED --> ACTIVE : Resolved
    ACTIVE --> ARCHIVED : Retire Permanently
    INACTIVE --> ARCHIVED : Retire Permanently
    ARCHIVED --> [*]
```

#### B. Multi-Dimensional Product Category Taxonomy (`category_dimensions`, `product_categories`, `product_category_assignments`)

Establishes a **modular, multi-dimensional taxonomy architecture** for structured tour classification, multi-faceted filtering, and SEO breadcrumb navigation. Categories answer: *"What type of trip is this?"* Products and variants are classified across 4 orthogonal dimensions simultaneously:

- **1. Taxonomy Dimensions (`category_dimensions`):**
  - **`TRAVEL_STYLE` (Format Operasional):** Operational tour format (*Paket Tour / Open Group Tour*, *Private Trip*, *Corporate & MICE*, *Signature / 5-Star Luxury*).
  - **`THEME_INTEREST` (Tema Wisata & Minat):** Core tour themes (*Cultural & Wonders*, *Nature & Alpine Scenery*, *Sakura & Flower Blooms*, *City Highlights & Shopping*, *Safari & Wildlife*).
  - **`SEASON_MOMENT` (Musim & Momen Liburan Peak):** Combines natural travel seasons (*Spring & Sakura Blossom*, *Summer Holiday*, *Autumn Leaves & Foliage*, *Winter Snow & Glacier*) and major Indonesian holiday peak windows (*NATARU / Year-End*, *Liburan Lebaran / Eid*, *Liburan Sekolah*).
  - **`SPECIAL_EXPERIENCE` (Preferensi Layanan & Dietary):** Dietary & demographic accommodations (*Halal Friendly / Muslim Tour*, *Family Friendly*, *Senior & Leisure Friendly*).

- **2. Hierarchical Categories (`product_categories`):**
  - Concrete category nodes linked to a dimension (`dimension_id`) with optional parent-child nesting (`parent_id`) within that dimension.

- **3. Modular Scoped Assignment (`product_category_assignments`):**
  - **Product-Level (`variant_id IS NULL`):** Global categories assigned to the master product umbrella (L1) are automatically inherited by all child variants (e.g. `Cultural & Wonders`, `Halal Friendly`, `Paket Tour / Open Group Tour`).
  - **Variant-Level (`variant_id IS NOT NULL`):** Variant-specific category tags (e.g. *Japan Sakura 7D* variant gets `Spring & Sakura Season` and `Sakura & Flower Blooms`, *Korea Autumn 6D* variant gets `Autumn Leaves & Foliage`, *Grand Europe Nataru* variant gets `NATARU / Year-End`).
  - **Primary Category per Dimension (`is_primary = TRUE`):** Flags the primary category per dimension for PDP breadcrumbs, meta tags, and category chip highlights.

#### C. Age Band Types & Quota Allocation (`age_band`) — `product_trip_pricings`

Replaces the legacy nationality scope (domestic and international prices are identical). Focuses on traveler age classification and dynamic seat capacity impact:

| `age_band` | Target Demographics | Quota / Seat Impact (`consumes_quota`) | Commercial Rationale |
| :--- | :--- | :--- | :--- |
| **`ADULT`** | Age 12+ years old (and standard bed occupancy) | Configurable `BOOLEAN` (Default `TRUE`) | Full adult rate, standard twin/double share accommodation and coach/flight seat. |
| **`INFANT`** | Under 2 years (< 24 months) | Configurable `BOOLEAN` (Default `FALSE`) | Surcharge/tax rate. May consume quota (`TRUE`) if seat/bed is allocated, or not (`FALSE`) if travelling as a lap infant. |

#### D. All-Inclusive Bundled Pricing, Itemized Component Breakdown & Excluded Add-ons

Hobiholidays enforces a structured, transparent separation across 3 pricing sub-domains:

1. **All-Inclusive Base Pricing & Itemized Breakdown (`product_trip_pricings` & `product_pricing_components`):**
   - The tier selling price (e.g. `ADULT = IDR 10.000.000`) represents the **bundled package rate**.
   - `product_pricing_components` provides the concrete itemized cost composition explaining to the customer and financial systems exactly what the Adult price covers besides the base departure cost:
     $$\text{selling\_price} = \text{base\_departure\_amount} + \sum_{i=1}^{n} \text{included\_component\_amount}_i$$
     *Real-World Example (GWE Summer Adult = IDR 10.000.000):*
     - **Biaya Keberangkatan & Land Tour:** IDR 9.350.000
     - **Schengen Visa Fee:** IDR 500.000 (`is_included = TRUE`)
     - **Airport Shuttle / Transfer:** IDR 100.000 (`is_included = TRUE`)
     - **Tour Leader & Driver Tipping:** IDR 50.000 (`is_included = TRUE`)
2. **Excluded Add-on Subsystem (`product_addons`):**
   - Elective, optional upgrades and additions (`SINGLE_ROOM`, `BAGGAGE`, `FLIGHT_UPGRADE`, `EXPERIENTIAL_TOUR`, `INSURANCE`, `VISA_EXPRESS`, `SPECIAL_MEAL`).
   - Symmetrically mirrors the Itinerary pattern: configured with a **Variant Master Default** (`trip_id IS NULL`), with optional **Trip Departure Overrides / Exclusives** (`trip_id IS NOT NULL`).
   - These items are **strictly excluded** from the base package price. Add-ons specify `applicable_age_band` (`ADULT`, `INFANT`, or `ALL`) and supplement the base price during booking checkout.
3. **Descriptive Narrative Inclusions & Exclusions (`product_supplementaries`):**
   - High-level qualitative bullet points (`INCLUDED`, `EXCLUDED`, `IMPORTANT_INFO`) rendered on PDP marketing overview tabs.

#### E. Variant Types (`variant_type`) — `product_variants`

Categorizes bookable cards surfaced on the **All Tours** storefront. In Hobiholidays, catalog listings are **strictly based on variants**:

| `variant_type`    | Architectural & Business Role                                                                                       | Real-World Example in Hobiholidays                                                  | Frontend UI Badge                       | Catalog Filter Tag         |
| ----------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------- | -------------------------- |
| **`STANDARD`**    | Core year-round package with regular recurring departures. Unaffected by specific seasonal or promotional gimmicks. | _GWE Classic All-Year_, _Grand Europe Signature 7D_                                 | None / `⭐ Classic` / `🔥 Best Seller`   | "Regular Packages"         |
| **`SEASONAL`**    | Tied strictly to natural seasons, weather changes, or regional climate windows (Spring, Summer, Autumn, Winter).    | _GWE Spring 2026_, _GWE Summer 2026_, _Swiss Winter Alps_                           | `🌸 Spring` / `☀️ Summer` / `❄️ Winter`  | "Spring / Autumn / Winter" |
| **`THEMED`**      | Centered around cultural festivals, flower blooms, sports events, or special attractions.                           | _GWE Tulip Keukenhof_, _Swiss Glacier Wonderland_, _Christmas Market Tour_          | `🌷 Tulip Edition` / `🎌 Festival`      | "Themed & Events"          |
| **`PROMOTIONAL`** | Limited-seat commercial releases, early bird launches, or flash sale campaigns with special pricing.                | _Early Bird Europe 2026_, _Flash Sale GWE IDR 24.9M_, _Travel Fair Special_         | `🔥 Flash Sale` / `⚡ Early Bird`       | "Promotions & Deals"       |

#### F. Product Types (`product_type`) — `products`

- **`JOURNEY`:** Flagship curated multi-day tour program.
- **`OPEN_TRIP`:** Scheduled open-registration group departure.
- **`PRIVATE_TRIP`:** Bespoke / custom private charter tour.
- **`DAY_TOUR`:** Single-day guided excursion or city tour.

#### G. Trip Departure Status (`status`) — `product_trips`

- **`ACTIVE`:** Open for booking; quota available.
- **`FULL`:** Sold out; maximum capacity reached.
- **`CANCELLED`:** Departure cancelled (minimum quota unmet or force majeure).
- **`COMPLETED`:** Tour concluded successfully.

#### H. Promotional Badges Subsystem (`product_badges` & `product_variant_badges`) vs Category Taxonomy

To keep the platform clean and prevent data duplication between structural taxonomy and marketing overlays:

| Dimension / Aspect | Category Taxonomy (`product_categories`) | Promotional Badges (`product_badges`) |
| :--- | :--- | :--- |
| **Core Question** | *"What is this tour package?"* | *"What promotional/editorial hook highlights this card?"* |
| **Primary Role** | Structured taxonomy, multi-faceted sidebar filters, breadcrumbs, and Schema.org SEO. | Visual UI ribbon/pill rendered on the top of variant card media and PDP hero. |
| **Key Examples** | `Cultural & Wonders`, `Halal Friendly`, `Spring & Sakura Season`, `Open Group Tour`. | `🔥 Best Seller`, `⚡ Flash Sale`, `⭐ Premium`, `🆕 Baru`, `⚡ Early Bird`. |
| **Visual Customization** | Icon URL, slug, sort order. | Custom `background_color` (hex), `text_color` (hex), `icon_url`, and `sort_order`. |
| **Cardinality** | M:N to `products` (L1) and `product_variants` (L2) with `is_primary`. | M:N to `product_variants` via `product_variant_badges`. |

---

## 🛠️ PostgreSQL DDL Schema

> This is the complete, authoritative schema for the Product domain. Tables are ordered by dependency (parents before children).

```sql
-- =========================================================================
-- 0. EXTENSIONS
-- =========================================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- Required for GIN text search indexes


-- =========================================================================
-- 1. TAXONOMY & CORE — L1: category_dimensions, product_categories, products & product_journeys
-- =========================================================================

-- Category Dimensions (Taxonomy facets: Travel Style, Theme, Season, Special Experience)
CREATE TABLE category_dimensions (
    id              UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    code            VARCHAR(50)  UNIQUE NOT NULL, -- e.g. 'TRAVEL_STYLE', 'THEME_INTEREST', 'SEASON_MOMENT', 'SPECIAL_EXPERIENCE'
    name            VARCHAR(100) NOT NULL,        -- e.g. 'Travel Style', 'Theme & Interest', 'Season & Moment', 'Special Experience / Dietary'
    description     TEXT,
    is_multi_select BOOLEAN      NOT NULL DEFAULT TRUE, -- UI filter hint (single-select vs multi-select facet)
    sort_order      INT          NOT NULL DEFAULT 0,
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_dimensions_code ON category_dimensions(code);

-- Product category nodes (Supports multi-dimensional grouping and 2-tier tree per dimension)
CREATE TABLE product_categories (
    id           UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    dimension_id UUID         NOT NULL REFERENCES category_dimensions(id) ON DELETE RESTRICT,
    parent_id    UUID         NULL REFERENCES product_categories(id) ON DELETE RESTRICT, -- Intra-dimension parent (for 2-tier tree e.g. Cultural & Wonders -> World Heritage)
    name         VARCHAR(100) NOT NULL,
    slug         VARCHAR(100) NOT NULL,
    description  TEXT,
    icon_url     VARCHAR(500),
    sort_order   INT          NOT NULL DEFAULT 0,
    is_active    BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_category_dimension_slug UNIQUE (dimension_id, slug),
    CONSTRAINT chk_category_depth CHECK (id <> parent_id)
);
CREATE INDEX idx_categories_dimension_id ON product_categories(dimension_id);
CREATE INDEX idx_categories_parent_id    ON product_categories(parent_id);
CREATE INDEX idx_categories_slug         ON product_categories(slug);

-- Master product entity — the brand/program umbrella
CREATE TABLE products (
    id                 UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_type       VARCHAR(50)  NOT NULL,                    -- JOURNEY | OPEN_TRIP | PRIVATE_TRIP | DAY_TOUR
    code               VARCHAR(100) UNIQUE NOT NULL,             -- e.g. GWE-MASTER
    name               VARCHAR(255) NOT NULL,                    -- e.g. Grand West Europe, Swiss Alpine Panorama
    slug               VARCHAR(255) UNIQUE NOT NULL,             -- e.g. grand-west-europe
    itinerary_pdf_url  VARCHAR(500),                             -- Default umbrella brochure URL generated by ATW
    listing_status     VARCHAR(50)  NOT NULL DEFAULT 'DRAFT',    -- DRAFT | PENDING_REVIEW | ACTIVE | INACTIVE | ARCHIVED | SUSPENDED
    created_at         TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at         TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at         TIMESTAMP    NULL,

    CONSTRAINT chk_products_listing_status CHECK (listing_status IN ('DRAFT', 'PENDING_REVIEW', 'ACTIVE', 'INACTIVE', 'ARCHIVED', 'SUSPENDED')),
    CONSTRAINT chk_products_type           CHECK (product_type IN ('JOURNEY', 'OPEN_TRIP', 'PRIVATE_TRIP', 'DAY_TOUR'))
);
CREATE INDEX idx_products_status          ON products(listing_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_name_trgm       ON products USING GIN (name gin_trgm_ops);
CREATE INDEX idx_products_slug_trgm       ON products USING GIN (slug gin_trgm_ops);

-- Base journey metadata (1:1 with products)
CREATE TABLE product_journeys (
    product_id          UUID        PRIMARY KEY REFERENCES products(id) ON DELETE RESTRICT,
    duration_days       INT         NOT NULL DEFAULT 1,
    duration_nights     INT         NOT NULL DEFAULT 0,
    created_at          TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_duration CHECK (duration_days >= 1 AND duration_nights >= 0)
);


-- =========================================================================
-- 2. HIERARCHY — L2: product_variants & product_category_assignments
-- One product → many variants. Each variant is strictly one card on All Tours.
-- Supports multi-category assignment at Product level (L1) and Variant level (L2).
-- =========================================================================
CREATE TABLE product_variants (
    id              UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id      UUID         NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    variant_type    VARCHAR(50)  NOT NULL DEFAULT 'STANDARD', -- STANDARD | SEASONAL | THEMED | PROMOTIONAL
    name            VARCHAR(255) NOT NULL,                    -- e.g. "GWE Spring 2026"
    slug            VARCHAR(255) UNIQUE NOT NULL,              -- e.g. "gwe-spring-2026"
    code            VARCHAR(100) UNIQUE NOT NULL,              -- e.g. "GWE-SPR-2026"

    -- Duration override: NULL means inherit from product_journeys via COALESCE
    duration_days   INT          NULL,
    duration_nights INT          NULL,

    -- ATW itinerary brochure URL: NULL means inherit from products.itinerary_pdf_url
    itinerary_pdf_url VARCHAR(500) NULL,

    listing_status  VARCHAR(50)  NOT NULL DEFAULT 'DRAFT',    -- DRAFT | PENDING_REVIEW | ACTIVE | INACTIVE | ARCHIVED | SUSPENDED
    created_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at      TIMESTAMP    NULL,

    CONSTRAINT chk_variants_variant_type   CHECK (variant_type IN ('STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL')),
    CONSTRAINT chk_variants_listing_status CHECK (listing_status IN ('DRAFT', 'PENDING_REVIEW', 'ACTIVE', 'INACTIVE', 'ARCHIVED', 'SUSPENDED'))
);
CREATE INDEX idx_variants_product_id   ON product_variants(product_id);
CREATE INDEX idx_variants_slug         ON product_variants(slug);
CREATE INDEX idx_variants_status       ON product_variants(listing_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_variants_name_trgm    ON product_variants USING GIN (name gin_trgm_ops);
CREATE INDEX idx_variants_slug_trgm    ON product_variants USING GIN (slug gin_trgm_ops);

-- Multi-dimensional category assignment table (Modular for both Products L1 & Variants L2)
CREATE TABLE product_category_assignments (
    id          UUID      PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID      NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    variant_id  UUID      NULL REFERENCES product_variants(id) ON DELETE RESTRICT, -- NULL = Product-level (applies to all variants), NOT NULL = Variant-specific specialized category override/tag
    category_id UUID      NOT NULL REFERENCES product_categories(id) ON DELETE RESTRICT,
    is_primary  BOOLEAN   NOT NULL DEFAULT FALSE, -- Flag primary category per dimension for breadcrumbs and primary category chip
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_product_category_assignment UNIQUE NULLS NOT DISTINCT (product_id, variant_id, category_id)
);
CREATE INDEX idx_cat_assign_product_id  ON product_category_assignments(product_id);
CREATE INDEX idx_cat_assign_variant_id  ON product_category_assignments(variant_id);
CREATE INDEX idx_cat_assign_category_id ON product_category_assignments(category_id);
CREATE INDEX idx_cat_assign_lookup      ON product_category_assignments(category_id, product_id, variant_id);

-- Flat badges table (supports admin-managed custom promotional labels on listing cards)
CREATE TABLE product_badges (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    code             VARCHAR(50)  UNIQUE NOT NULL, -- e.g. 'BEST_SELLER', 'SPRING_EDITION'
    label            VARCHAR(100) NOT NULL,        -- e.g. '🔥 Best Seller', '🌸 Spring Edition'
    background_color VARCHAR(30)  DEFAULT '#F3F4F6',
    text_color       VARCHAR(30)  DEFAULT '#1F2937',
    icon_url         VARCHAR(500) NULL,
    is_active        BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Flat M:N mapping between variants and badges
CREATE TABLE product_variant_badges (
    variant_id UUID NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
    badge_id   UUID NOT NULL REFERENCES product_badges(id) ON DELETE RESTRICT,
    PRIMARY KEY (variant_id, badge_id)
);
CREATE INDEX idx_variant_badges_badge_id ON product_variant_badges(badge_id);


-- =========================================================================
-- 3. HIERARCHY — L3: product_trips, product_trip_pricings & components
-- One variant → many trips. A trip is a concrete dated departure window.
-- =========================================================================
CREATE TABLE product_trips (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    variant_id      UUID        NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
    start_date      DATE        NOT NULL,
    end_date        DATE        NOT NULL,
    min_quota       INT         NOT NULL DEFAULT 1,
    max_quota       INT         NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',    -- ACTIVE | FULL | CANCELLED | COMPLETED
    created_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_trip_variant_start UNIQUE (variant_id, start_date),
    CONSTRAINT chk_trip_dates        CHECK  (end_date > start_date),
    CONSTRAINT chk_trip_quota        CHECK  (max_quota >= min_quota AND min_quota >= 1),
    CONSTRAINT chk_trips_status      CHECK  (status IN ('ACTIVE', 'FULL', 'CANCELLED', 'COMPLETED'))
);
CREATE INDEX idx_trips_variant_id ON product_trips(variant_id);
-- Partial index: search queries only touch ACTIVE trips
CREATE INDEX idx_trips_search     ON product_trips(start_date, min_quota, max_quota)
    WHERE status = 'ACTIVE';

-- Pricing tiers per trip resolved by age band & capacity quota rules (replaces nationality)
CREATE TABLE product_trip_pricings (
    id                  UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    trip_id             UUID           NOT NULL REFERENCES product_trips(id) ON DELETE RESTRICT,
    age_band            VARCHAR(50)    NOT NULL,               -- ADULT | INFANT
    min_age             INT            NOT NULL DEFAULT 0,
    max_age             INT            NULL,
    consumes_quota      BOOLEAN        NOT NULL DEFAULT TRUE,  -- Configurable: TRUE if passenger occupies seat quota, FALSE if non-quota (e.g. lap infant)
    currency            VARCHAR(10)    NOT NULL DEFAULT 'IDR',
    base_price          DECIMAL(15,2)  NOT NULL,
    selling_price       DECIMAL(15,2)  NOT NULL,
    created_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_pricing_trip_age_band UNIQUE (trip_id, age_band),
    CONSTRAINT chk_price_sanity         CHECK  (selling_price > 0 AND base_price >= selling_price),
    CONSTRAINT chk_pricing_age_band     CHECK  (age_band IN ('ADULT', 'INFANT'))
);
CREATE INDEX idx_pricings_search ON product_trip_pricings(trip_id, age_band, selling_price);

-- Itemized pricing breakdown components (bundles included within tier selling_price: Visa, Shuttle, Tipping, etc.)
CREATE TABLE product_pricing_components (
    id          UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    pricing_id  UUID           NOT NULL REFERENCES product_trip_pricings(id) ON DELETE RESTRICT,
    name        VARCHAR(150)   NOT NULL, -- e.g. "Schengen Visa Fee", "Airport Shuttle & Coach", "Tour Leader & Driver Tipping"
    description TEXT,
    amount      DECIMAL(15,2)  NOT NULL DEFAULT 0.00,
    is_included BOOLEAN        NOT NULL DEFAULT TRUE, -- TRUE = bundled inside selling_price
    sort_order  INT            NOT NULL DEFAULT 0,
    created_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_component_amount CHECK (amount >= 0)
);
CREATE INDEX idx_pricing_components_pricing_id ON product_pricing_components(pricing_id);

-- Optional add-ons (Single Supplement, Extra Baggage, Flight Upgrade, Excursion)
-- Excluded from base price; available for elective passenger purchase
CREATE TABLE product_addons (
    id                  UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    variant_id          UUID           NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
    trip_id             UUID           NULL REFERENCES product_trips(id) ON DELETE SET NULL, -- NULL = all trips in variant
    code                VARCHAR(50)    NOT NULL, -- e.g. 'ADDON-SINGLE-SUPP'
    name                VARCHAR(255)   NOT NULL, -- e.g. "Single Supplement (Kamar Sendiri)"
    description         TEXT,
    addon_type          VARCHAR(50)    NOT NULL, -- SINGLE_ROOM | BAGGAGE | FLIGHT_UPGRADE | EXPERIENTIAL_TOUR | INSURANCE | VISA_EXPRESS | SPECIAL_MEAL
    charge_type         VARCHAR(50)    NOT NULL DEFAULT 'PER_PAX', -- PER_PAX | PER_ROOM | PER_BOOKING
    price               DECIMAL(15,2)  NOT NULL,
    currency            VARCHAR(10)    NOT NULL DEFAULT 'IDR',
    applicable_age_band VARCHAR(20)    NULL, -- NULL = ALL | ADULT | INFANT
    is_mandatory        BOOLEAN        NOT NULL DEFAULT FALSE,
    max_quantity        INT            NOT NULL DEFAULT 1,
    is_active           BOOLEAN        NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_addon_price       CHECK (price >= 0),
    CONSTRAINT chk_addon_charge_type CHECK (charge_type IN ('PER_PAX', 'PER_ROOM', 'PER_BOOKING')),
    CONSTRAINT chk_addon_type        CHECK (addon_type IN ('SINGLE_ROOM', 'BAGGAGE', 'FLIGHT_UPGRADE', 'EXPERIENTIAL_TOUR', 'INSURANCE', 'VISA_EXPRESS', 'SPECIAL_MEAL')),
    CONSTRAINT chk_addon_age_band    CHECK (applicable_age_band IS NULL OR applicable_age_band IN ('ADULT', 'INFANT'))
);
CREATE INDEX idx_addons_variant_trip ON product_addons(variant_id, trip_id);

-- Variant default add-on: exactly 1 master default per code per variant where trip_id IS NULL
CREATE UNIQUE INDEX uq_addon_variant_code 
    ON product_addons (variant_id, code) 
    WHERE trip_id IS NULL;

-- Trip override / exclusive add-on: at most 1 override/exclusive per code per specific departure
CREATE UNIQUE INDEX uq_addon_trip_code 
    ON product_addons (trip_id, code) 
    WHERE trip_id IS NOT NULL;

-- Canonical Fallback Resolution Query:
-- resolved_addon(code) = trip.addon(code) ?? variant.addon(code)
-- SELECT * FROM (
--     SELECT DISTINCT ON (code) *
--     FROM product_addons
--     WHERE trip_id = :tripId OR (variant_id = :variantId AND trip_id IS NULL)
--     ORDER BY code, trip_id ASC NULLS LAST
-- ) resolved_addons
-- WHERE is_active = TRUE
-- ORDER BY code ASC;


-- =========================================================================
-- 4. CONTENT — Itinerary (Variant Level Default & Trip Level Override)
-- Day-by-day programme. Default owned at Variant (L2), overridden at Trip (L3).
-- =========================================================================
CREATE TABLE product_itineraries (
    id              UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    variant_id      UUID         NOT NULL REFERENCES product_variants(id) ON DELETE RESTRICT,
    trip_id         UUID         NULL REFERENCES product_trips(id) ON DELETE SET NULL,
    source_type     VARCHAR(50)  NOT NULL DEFAULT 'INTERNAL', -- MERCHANT | INTERNAL
    itinerary_type  VARCHAR(50)  NOT NULL DEFAULT 'STANDARD', -- STANDARD | CUSTOM
    title           VARCHAR(255) NOT NULL,
    summary         TEXT,
    created_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_itineraries_source_type CHECK (source_type IN ('MERCHANT', 'INTERNAL')),
    CONSTRAINT chk_itineraries_type        CHECK (itinerary_type IN ('STANDARD', 'CUSTOM'))
);

-- Variant default itinerary: exactly 1 master itinerary per variant where trip_id IS NULL
CREATE UNIQUE INDEX uq_itinerary_variant_default 
    ON product_itineraries (variant_id) 
    WHERE trip_id IS NULL;

-- Trip override itinerary: at most 1 custom itinerary per specific departure
CREATE UNIQUE INDEX uq_itinerary_trip_override 
    ON product_itineraries (trip_id) 
    WHERE trip_id IS NOT NULL;

CREATE TABLE product_itinerary_items (
    id              UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    itinerary_id    UUID         NOT NULL REFERENCES product_itineraries(id) ON DELETE RESTRICT,
    day_number      INT          NOT NULL,
    sequence_number INT          NOT NULL,
    item_type       VARCHAR(50)  NOT NULL,       -- ACTIVITY | TRANSPORT | MEAL | ACCOMMODATION | OTHER
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    poi_area_id     UUID         NULL,           -- Logical FK → areas.id (POI landmark)
    meals_included  VARCHAR(100),
    accommodation   VARCHAR(150),
    created_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_itinerary_item_order UNIQUE (itinerary_id, day_number, sequence_number),
    CONSTRAINT chk_itinerary_item_type  CHECK (item_type IN ('ACTIVITY', 'TRANSPORT', 'MEAL', 'ACCOMMODATION', 'OTHER'))
);
CREATE INDEX idx_itinerary_items_itinerary_id ON product_itinerary_items(itinerary_id);
CREATE INDEX idx_itinerary_items_poi          ON product_itinerary_items(poi_area_id);


-- =========================================================================
-- 5. CONTENT — Locations
-- Multiple destination markers per product. area_id is a logical FK to the
-- Area/Geography domain (anchored to any tier: POI, COUNTRY, SUB_CONTINENT, or CONTINENT).
-- Leaf tiers below the linked anchor evaluate to NULL in flat DTO responses.
-- =========================================================================
CREATE TABLE product_locations (
    id          UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID           NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    source_type VARCHAR(50)    NOT NULL,                   -- AREA | MANUAL
    area_id     UUID           NOT NULL,                   -- Logical FK → areas.id (POI, COUNTRY, SUB_CONTINENT, or CONTINENT)
    area_name   VARCHAR(100),                              -- Denormalized for fast UI rendering
    address     TEXT,
    sort_order  INT            NOT NULL DEFAULT 0,
    created_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_locations_source_type CHECK (source_type IN ('AREA', 'MANUAL'))
);
CREATE INDEX idx_locations_product_id     ON product_locations(product_id);
CREATE INDEX idx_locations_area_id        ON product_locations(area_id);
CREATE INDEX idx_locations_area_name_trgm ON product_locations USING GIN (area_name gin_trgm_ops);


-- =========================================================================
-- 6. MEDIA — product_media & product_media_usages (polymorphic)
-- Media assets are owned at the product level.
-- Usage slots (cover, gallery, thumbnail) target entities polymorphically.
-- Supports visual marketing assets (images and videos). Tour PDF brochures are compiled by ATW.
-- Detailed Architecture: See [Product Media Technical Design](./product-media-technical-design.md)
-- =========================================================================
CREATE TABLE product_media (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id       UUID         NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    storage_provider VARCHAR(50)  NOT NULL DEFAULT 'DATABASE', -- DATABASE (Phase 1) | S3 | CLOUDFLARE_R2 (Phase 2)
    source_upload_id VARCHAR(255),                          -- External upload service or CDN reference ID
    media_type       VARCHAR(50)  NOT NULL,                  -- IMAGE | VIDEO
    file_name        VARCHAR(255) NOT NULL,                 -- Original filename (e.g. gwe-eiffel-tower-hero.jpg)
    file_size_bytes  BIGINT       NOT NULL,                 -- File size in bytes for UI preview
    mime_type        VARCHAR(100) NOT NULL,                 -- e.g. image/jpeg, image/webp, video/mp4
    object_key       VARCHAR(500) NULL,                     -- S3/GCS key (NULL in Phase 1, required in Phase 2)
    url              VARCHAR(500) NOT NULL,                 -- Stream URL (Phase 1) or CDN URL (Phase 2)
    created_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_media_type             CHECK (media_type IN ('IMAGE', 'VIDEO')),
    CONSTRAINT chk_media_storage_provider CHECK (storage_provider IN ('DATABASE', 'S3', 'CLOUDFLARE_R2'))
);
CREATE INDEX idx_media_product_id ON product_media(product_id);
CREATE INDEX idx_media_type       ON product_media(media_type);
CREATE INDEX idx_media_storage    ON product_media(storage_provider);

-- Dedicated table for Phase 1 in-database binary storage (BYTEA)
CREATE TABLE product_media_blobs (
    media_id   UUID      PRIMARY KEY REFERENCES product_media(id) ON DELETE RESTRICT,
    file_data  BYTEA     NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_media_usages (
    id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    media_id      UUID        NOT NULL REFERENCES product_media(id) ON DELETE RESTRICT,
    target_type   VARCHAR(50) NOT NULL,    -- PRODUCT | VARIANT | ITINERARY_ITEM
    target_id     UUID        NOT NULL,    -- Polymorphic — resolved by target_type
    usage_context VARCHAR(50) NOT NULL,    -- COVER | GALLERY | THUMBNAIL | ATTACHMENT
    sort_order    INT         NOT NULL DEFAULT 0,
    created_at    TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_media_target_type    CHECK (target_type IN ('PRODUCT', 'VARIANT', 'ITINERARY_ITEM')),
    CONSTRAINT chk_media_usage_context CHECK (usage_context IN ('COVER', 'GALLERY', 'THUMBNAIL', 'ATTACHMENT'))
);
-- Essential composite index for polymorphic reads
CREATE INDEX idx_media_usages_target ON product_media_usages(target_type, target_id);

-- Enforce single COVER image per entity target
CREATE UNIQUE INDEX uq_media_usages_single_cover
    ON product_media_usages(target_id, usage_context)
    WHERE usage_context = 'COVER';


-- =========================================================================
-- 7. SUPPLEMENTARY CONTENT (polymorphic)
-- Reusable content blocks that can target any entity level.
-- =========================================================================
CREATE TABLE product_supplementaries (
    id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID        NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    target_type VARCHAR(50) NOT NULL,    -- PRODUCT | VARIANT | TRIP
    target_id   UUID        NOT NULL,    -- Polymorphic
    category    VARCHAR(50) NOT NULL,    -- INCLUDED | EXCLUDED | IMPORTANT_INFO | NOTE
    content     TEXT,
    sort_order  INT         NOT NULL DEFAULT 0,
    created_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_supp_target_type CHECK (target_type IN ('PRODUCT', 'VARIANT', 'TRIP')),
    CONSTRAINT chk_supp_category    CHECK (category IN ('INCLUDED', 'EXCLUDED', 'IMPORTANT_INFO', 'NOTE'))
);
CREATE INDEX idx_supplementaries_target ON product_supplementaries(target_type, target_id);


-- =========================================================================
-- 8. SEO METADATA (polymorphic)
-- Custom search engine optimization and Open Graph tags per entity.
-- Detailed Architecture: See [SEO Technical Design](./seo-technical-design.md)
-- =========================================================================
CREATE TABLE seo_metadata (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    target_type      VARCHAR(50)  NOT NULL,                  -- PRODUCT | VARIANT | AREA
    target_id        UUID         NOT NULL,                  -- Polymorphic FK
    meta_title       VARCHAR(255),                          -- Custom <title> tag
    meta_description TEXT,                                  -- Custom meta description
    canonical_url    VARCHAR(500),                          -- Canonical URL override
    og_title         VARCHAR(255),                          -- Open Graph title
    og_description   TEXT,                                  -- Open Graph description
    og_image_url     VARCHAR(500),                          -- Social sharing banner
    no_index         BOOLEAN      NOT NULL DEFAULT FALSE,   -- Crawler noindex flag
    no_follow        BOOLEAN      NOT NULL DEFAULT FALSE,   -- Crawler nofollow flag
    structured_data  JSONB,                                 -- Schema.org overrides
    created_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_seo_target_type CHECK (target_type IN ('PRODUCT', 'VARIANT', 'AREA')),
    CONSTRAINT uq_seo_target       UNIQUE (target_type, target_id)
);
CREATE INDEX idx_seo_target ON seo_metadata(target_type, target_id);


-- =========================================================================
-- 9. AUDIT TRIGGER AUTOMATION
-- PostgreSQL trigger function to automatically update `updated_at` timestamps
-- upon row mutation across all domain entities.
-- =========================================================================
CREATE OR REPLACE FUNCTION set_updated_at_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply timestamp triggers to all tables with updated_at
CREATE TRIGGER trg_category_dimensions_updated_at BEFORE UPDATE ON category_dimensions   FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_categories_updated_at  BEFORE UPDATE ON product_categories    FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_products_updated_at             BEFORE UPDATE ON products             FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_journeys_updated_at     BEFORE UPDATE ON product_journeys     FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_variants_updated_at     BEFORE UPDATE ON product_variants     FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_trips_updated_at        BEFORE UPDATE ON product_trips        FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_trip_pricings_updated_at BEFORE UPDATE ON product_trip_pricings FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_pricing_components_updated_at BEFORE UPDATE ON product_pricing_components FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_itineraries_updated_at  BEFORE UPDATE ON product_itineraries  FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_itinerary_items_updated_at BEFORE UPDATE ON product_itinerary_items FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_locations_updated_at    BEFORE UPDATE ON product_locations    FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_media_updated_at        BEFORE UPDATE ON product_media        FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_media_usages_updated_at BEFORE UPDATE ON product_media_usages FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_supplementaries_updated_at BEFORE UPDATE ON product_supplementaries FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_seo_metadata_updated_at         BEFORE UPDATE ON seo_metadata         FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
```

---

## 📊 Domain Data Scenario — Grand West Europe

**Product:** Grand West Europe · **ID:** `prod_gwe_01`

_Each section below shows the ERD for that sub-domain, followed by concrete sample rows. (Standard audit timestamps `created_at`, `updated_at`, and `deleted_at` are defined in the schema and ERD above, but omitted from the sample data tables below for readability)._

---

### 1. Core Product, Taxonomy & Multi-Dimensional Category Assignments

```mermaid
erDiagram
    category_dimensions ||--o{ product_categories : "dimension_id"
    product_categories  ||--o{ product_categories : "parent_id"
    products            ||--o| product_journeys   : "product_id (1:1)"
    products            ||--o{ product_category_assignments : "product_id (L1)"
    product_variants    ||--o{ product_category_assignments : "variant_id (L2)"
    product_categories  ||--o{ product_category_assignments : "category_id"

    category_dimensions {
        uuid      id              PK
        varchar   code            "TRAVEL_STYLE | THEME_INTEREST | SEASON_MOMENT | SPECIAL_EXPERIENCE"
        varchar   name            "Travel Style, Theme & Interest, etc."
        boolean   is_multi_select
    }

    product_categories {
        uuid      id           PK
        uuid      dimension_id FK
        uuid      parent_id    FK "self-reference within dimension"
        varchar   name         "e.g. Popular Group Tours, Cultural & Heritage, Halal Friendly"
        varchar   slug         "e.g. popular-group-tours, cultural-heritage"
    }

    product_category_assignments {
        uuid      id          PK
        uuid      product_id  FK "products.id"
        uuid      variant_id  FK "product_variants.id (NULL = product-level)"
        uuid      category_id FK "product_categories.id"
        boolean   is_primary  "Flags primary category per dimension"
    }

    products {
        uuid      id                 PK
        varchar   product_type       "JOURNEY | OPEN_TRIP | PRIVATE_TRIP | DAY_TOUR"
        varchar   code
        varchar   name
        varchar   slug
        varchar   itinerary_pdf_url  "ATW default brochure PDF"
        varchar   listing_status     "DRAFT | PENDING_REVIEW | ACTIVE | INACTIVE | ARCHIVED | SUSPENDED"
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    product_journeys {
        uuid      product_id        PK "FK → products"
        int       duration_days
        int       duration_nights
        timestamp created_at
        timestamp updated_at
    }
```

#### Sample `category_dimensions`

| id | code | name | is_multi_select | sort_order |
| :--- | :--- | :--- | :--- | :--- |
| `dim_travel_style` | `TRAVEL_STYLE` | Format Operasional (Travel Style) | FALSE | 1 |
| `dim_theme` | `THEME_INTEREST` | Tema Wisata & Minat | TRUE | 2 |
| `dim_season` | `SEASON_MOMENT` | Musim & Momen Liburan (Holiday Peak) | TRUE | 3 |
| `dim_special` | `SPECIAL_EXPERIENCE` | Preferensi Layanan & Dietary | TRUE | 4 |

#### Sample `product_categories`

| id | dimension_id | parent_id | name | slug |
| :--- | :--- | :--- | :--- | :--- |
| `cat_travel_style` | `dim_travel_style` | NULL | Travel Style | travel-style |
| `cat_pop_group_tours` | `dim_travel_style` | `cat_travel_style` | Popular Group Tours | popular-group-tours |
| `cat_private_trip` | `dim_travel_style` | `cat_travel_style` | Private Trip | private-trip |
| `cat_corporate_mice` | `dim_travel_style` | `cat_travel_style` | Corporate & MICE | corporate-mice |
| `cat_signature_premium` | `dim_travel_style` | `cat_travel_style` | Signature 5-Star Tour | signature-5star-tour |
| `cat_cultural_wonders` | `dim_theme` | NULL | Cultural & Wonders | cultural-wonders |
| `cat_nature_scenic` | `dim_theme` | NULL | Nature & Alpine Scenery | nature-alpine-scenery |
| `cat_flower_bloom` | `dim_theme` | NULL | Sakura & Flower Blooms | sakura-flower-blooms |
| `cat_city_shopping` | `dim_theme` | NULL | City Highlights & Shopping | city-highlights-shopping |
| `cat_safari_wildlife` | `dim_theme` | NULL | Safari & Wildlife | safari-wildlife |
| `cat_spring_sakura` | `dim_season` | NULL | Spring & Sakura Season | spring-sakura-season |
| `cat_summer_holiday` | `dim_season` | NULL | Summer Holiday | summer-holiday |
| `cat_autumn_foliage` | `dim_season` | NULL | Autumn Leaves & Foliage | autumn-foliage-leaves |
| `cat_winter_snow` | `dim_season` | NULL | Winter Snow & Glacier | winter-snow-glacier |
| `cat_nataru` | `dim_season` | NULL | Natal & Tahun Baru (NATARU) | nataru-year-end |
| `cat_libur_lebaran` | `dim_season` | NULL | Liburan Lebaran (Eid) | liburan-lebaran-eid |
| `cat_libur_sekolah` | `dim_season` | NULL | Liburan Sekolah | liburan-sekolah |
| `cat_halal_friendly` | `dim_special` | NULL | Halal / Muslim Friendly | halal-muslim-friendly |
| `cat_family_friendly` | `dim_special` | NULL | Family Friendly | family-friendly |
| `cat_senior_friendly` | `dim_special` | NULL | Senior & Leisure Friendly | senior-leisure-friendly |

#### Sample `products` & `product_journeys`

| Table | id | product_type | code | slug | listing_status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `products` | `prod_gwe_01` | JOURNEY | GWE-MASTER | grand-west-europe | ACTIVE |
| `products` | `prod_jpn_01` | JOURNEY | JPN-MASTER | japan-golden-route | ACTIVE |
| `products` | `prod_kor_01` | JOURNEY | KOR-MASTER | korea-autumn | ACTIVE |
| `products` | `prod_tur_01` | JOURNEY | TUR-MASTER | turkey-wonders | ACTIVE |
| `products` | `prod_sws_01` | JOURNEY | SWS-MASTER | swiss-alps-signature | ACTIVE |
| `products` | `prod_afr_01` | JOURNEY | AFR-MASTER | egypt-morocco | ACTIVE |
| `products` | `prod_anz_01` | JOURNEY | ANZ-MASTER | aussie-nz-explorer | ACTIVE |
| `products` | `prod_usa_01` | JOURNEY | USA-MASTER | usa-west-coast | ACTIVE |

| Table | product_id | duration_days | duration_nights |
| :--- | :--- | :--- | :--- |
| `product_journeys` | `prod_gwe_01` | 7 | 6 |
| `product_journeys` | `prod_jpn_01` | 7 | 5 |
| `product_journeys` | `prod_kor_01` | 6 | 4 |
| `product_journeys` | `prod_tur_01` | 9 | 7 |
| `product_journeys` | `prod_sws_01` | 8 | 6 |
| `product_journeys` | `prod_afr_01` | 12 | 10 |
| `product_journeys` | `prod_anz_01` | 10 | 8 |
| `product_journeys` | `prod_usa_01` | 11 | 9 |

#### Sample `product_category_assignments` (Multi-Dimensional & Multi-Level)

| id | product_id | variant_id | category_id | Category Name | Dimension | is_primary | Assignment Scope |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `pca_01` | `prod_gwe_01` | **NULL** | `cat_pop_group_tours` | Popular Group Tours | `TRAVEL_STYLE` | **TRUE** | **L1: Product (Grand West Europe)** |
| `pca_02` | `prod_gwe_01` | **NULL** | `cat_cultural_wonders` | Cultural & Wonders | `THEME_INTEREST` | **TRUE** | **L1: Product (Grand West Europe)** |
| `pca_03` | `prod_gwe_01` | **NULL** | `cat_halal_friendly` | Halal / Muslim Friendly | `SPECIAL_EXPERIENCE` | **TRUE** | **L1: Product (Grand West Europe)** |
| `pca_04` | `prod_gwe_01` | `var_gwe_std_26` | `cat_cultural_wonders` | Cultural & Wonders | `THEME_INTEREST` | **TRUE** | **L2: Variant (`var_gwe_std_26` Classic All-Year)** |
| `pca_05` | `prod_gwe_01` | `var_gwe_spr_26` | `cat_spring_sakura` | Spring & Sakura Season | `SEASON_MOMENT` | **TRUE** | **L2: Variant (`var_gwe_spr_26` Spring 2026)** |
| `pca_06` | `prod_gwe_01` | `var_gwe_sum_26` | `cat_summer_holiday` | Summer Holiday | `SEASON_MOMENT` | **TRUE** | **L2: Variant (`var_gwe_sum_26` Summer 2026)** |
| `pca_07` | `prod_gwe_01` | `var_gwe_tlp_26` | `cat_spring_sakura` | Spring & Sakura Season | `SEASON_MOMENT` | FALSE | **L2: Variant (`var_gwe_tlp_26` Tulip Keukenhof)** |
| `pca_08` | `prod_gwe_01` | `var_gwe_tlp_26` | `cat_flower_bloom` | Sakura & Flower Blooms | `THEME_INTEREST` | **TRUE** | **L2: Variant (`var_gwe_tlp_26` Tulip Keukenhof)** |
| `pca_09` | `prod_gwe_01` | `var_gwe_eb_26` | `cat_cultural_wonders` | Cultural & Wonders | `THEME_INTEREST` | **TRUE** | **L2: Variant (`var_gwe_eb_26` Early Bird Europe)** |
| `pca_10` | `prod_jpn_01` | `var_jpn_sakura` | `cat_flower_bloom` | Sakura & Flower Blooms | `THEME_INTEREST` | **TRUE** | **L2: Variant (`var_jpn_sakura` Japan Sakura 7D)** |
| `pca_11` | `prod_sws_01` | `var_sws_sig` | `cat_signature_premium`| Signature 5-Star Tour | `TRAVEL_STYLE` | **TRUE** | **L2: Variant (`var_sws_sig` Swiss Alps Signature)** |

#### Sample `product_badges` (Marketing Visual Card Ribbons)

| id | code | label | background_color | text_color | icon_url | is_active |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `badge_best_seller` | `BEST_SELLER` | 🔥 Best Seller | `#004FC0` | `#FFFFFF` | NULL | TRUE |
| `badge_spring` | `SPRING_EDITION` | 🌸 Spring Edition | `#FDF2F8` | `#9D174D` | NULL | TRUE |
| `badge_summer` | `SUMMER_HOLIDAY` | ☀️ Summer Holiday | `#FEF3C7` | `#92400E` | NULL | TRUE |
| `badge_tulip` | `TULIP_SPECIAL` | 🌷 Tulip Edition | `#F0FDF4` | `#166534` | NULL | TRUE |
| `badge_early_bird` | `EARLY_BIRD` | ⚡ Early Bird | `#FFA80F` | `#0A1426` | NULL | TRUE |
| `badge_flash_sale` | `FLASH_SALE` | ⚡ Flash Sale | `#E8352A` | `#FFFFFF` | NULL | TRUE |
| `badge_populer` | `POPULER` | ✨ Populer | `#FFA80F` | `#0A1426` | NULL | TRUE |
| `badge_premium` | `PREMIUM` | ⭐ Premium | `#0A1426` | `#FFA80F` | NULL | TRUE |
| `badge_baru` | `BARU` | 🆕 Baru | `#188a42` | `#FFFFFF` | NULL | TRUE |

#### Sample `product_variant_badges` (M:N Badges to Variants)

| variant_id | badge_id | Applied Variant Card | Visual Rendering on Storefront |
| :--- | :--- | :--- | :--- |
| `var_gwe_std_26` | `badge_best_seller` | GWE Classic All-Year | Blue pill `🔥 Best Seller` on top-left card thumbnail & PDP header |
| `var_gwe_spr_26` | `badge_spring` | GWE Spring 2026 | Pink pill `🌸 Spring Edition` |
| `var_gwe_sum_26` | `badge_summer` | GWE Summer 2026 | Yellow pill `☀️ Summer Holiday` |
| `var_gwe_tlp_26` | `badge_tulip` | GWE Tulip Keukenhof | Green pill `🌷 Tulip Edition` |
| `var_gwe_eb_26` | `badge_early_bird` | Early Bird Europe | Orange pill `⚡ Early Bird` |
| `var_jpn_sakura` | `badge_populer` | Japan Sakura Golden Route | Orange pill `✨ Populer` |
| `var_sws_sig` | `badge_premium` | Swiss Alps Signature 8D | Dark navy pill with gold text `⭐ Premium` |
| `var_afr_egy_mor` | `badge_baru` | Egypt & Morocco 12D | Green pill `🆕 Baru` |

---

### 2. Variants, Trips, Age-Band Pricing, Components & Add-ons

> Full hierarchy ERDs and GWE sample data: [Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)

```mermaid
erDiagram
    products              ||--o{ product_variants          : "product_id"
    product_variants      ||--o{ product_trips             : "variant_id"
    product_trips         ||--o{ product_trip_pricings      : "trip_id"
    product_trip_pricings ||--o{ product_pricing_components: "pricing_id"
    product_variants      ||--o{ product_addons            : "variant_id (default master extras)"
    product_trips         ||--o{ product_addons            : "trip_id (trip override / exclusive)"
    product_variants      ||--o{ product_variant_badges    : "variant_id"
    product_badges        ||--o{ product_variant_badges    : "badge_id"

    product_variants {
        uuid      id                 PK
        uuid      product_id         FK
        varchar   variant_type       "STANDARD | SEASONAL | THEMED | PROMOTIONAL"
        varchar   name
        varchar   slug
        varchar   code
        varchar   itinerary_pdf_url  "ATW variant brochure PDF"
        int       duration_days      "NULL = inherit"
        int       duration_nights    "NULL = inherit"
        varchar   listing_status     "ACTIVE"
    }

    product_badges {
        uuid      id                 PK
        varchar   code               "BEST_SELLER | FLASH_SALE | EARLY_BIRD | POPULER | PREMIUM | BARU"
        varchar   label              "🔥 Best Seller"
        varchar   background_color
        varchar   text_color
        boolean   is_active
    }

    product_trips {
        uuid      id         PK
        uuid      variant_id FK
        date      start_date
        date      end_date
        int       min_quota
        int       max_quota
        varchar   status     "ACTIVE | FULL | CANCELLED | COMPLETED"
    }

    product_trip_pricings {
        uuid       id             PK
        uuid       trip_id        FK
        varchar    age_band       "ADULT | INFANT"
        boolean    consumes_quota "true | false (infant may use quota)"
        decimal    base_price
        decimal    selling_price
    }

    product_pricing_components {
        uuid       id          PK
        uuid       pricing_id  FK
        varchar    name        "Schengen Visa Fee, Airport Shuttle, Tip, etc."
        decimal    amount      "Included amount in selling_price"
        boolean    is_included "true"
        int        sort_order
    }

    product_addons {
        uuid       id                  PK
        uuid       variant_id          FK
        uuid       trip_id             FK
        varchar    code                "ADDON-SINGLE-SUPP"
        varchar    name                "Single Supplement"
        varchar    addon_type          "SINGLE_ROOM | BAGGAGE | FLIGHT_UPGRADE | EXPERIENTIAL_TOUR | INSURANCE | VISA_EXPRESS | SPECIAL_MEAL"
        varchar    charge_type         "PER_PAX | PER_ROOM | PER_BOOKING"
        varchar    applicable_age_band "ALL | ADULT | INFANT"
        decimal    price
        boolean    is_mandatory        "false | true"
        int        max_quantity        "1"
        boolean    is_active           "true"
    }
```

| Table | id | product_id | variant_type | name | slug | code | duration_days | duration_nights | listing_status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `product_variants` | var_gwe_std_26 | prod_gwe_01 | STANDARD | GWE Classic All-Year | gwe-classic-all-year | GWE-STD-2026 | NULL (7) | NULL (6) | ACTIVE |
| `product_variants` | var_gwe_spr_26 | prod_gwe_01 | SEASONAL | GWE Spring 2026 | gwe-spring-2026 | GWE-SPR-2026 | NULL (7) | NULL (6) | ACTIVE |
| `product_variants` | var_gwe_sum_26 | prod_gwe_01 | SEASONAL | GWE Summer 2026 | gwe-summer-2026 | GWE-SUM-2026 | NULL (7) | NULL (6) | ACTIVE |
| `product_variants` | var_gwe_tlp_26 | prod_gwe_01 | THEMED | GWE Tulip Keukenhof | gwe-tulip-keukenhof | GWE-TLP-2026 | NULL (7) | NULL (6) | ACTIVE |
| `product_variants` | var_gwe_eb_26 | prod_gwe_01 | PROMOTIONAL | Early Bird Europe | early-bird-europe-2026 | GWE-EB-2026 | NULL (7) | NULL (6) | ACTIVE |

| Table | id | variant_id | start_date | end_date | min_quota | max_quota | status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `product_trips` | trip_gwe_std_01 | var_gwe_std_26 | 2026-08-05 | 2026-08-11 | 15 | 30 | ACTIVE |
| `product_trips` | trip_gwe_spr_01 | var_gwe_spr_26 | 2026-09-10 | 2026-09-16 | 15 | 30 | ACTIVE |
| `product_trips` | trip_gwe_sum_01 | var_gwe_sum_26 | 2026-07-10 | 2026-07-16 | 20 | 35 | ACTIVE |
| `product_trips` | trip_gwe_tlp_01 | var_gwe_tlp_26 | 2026-04-15 | 2026-04-21 | 15 | 25 | ACTIVE |
| `product_trips` | trip_gwe_eb_01 | var_gwe_eb_26 | 2026-11-01 | 2026-11-07 | 10 | 20 | ACTIVE |

| Table | id | trip_id | age_band | consumes_quota | base_price | selling_price |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `product_trip_pricings` | pricing_std_ad | trip_gwe_std_01 | ADULT | TRUE | 32000000.00 | 28500000.00 |
| `product_trip_pricings` | pricing_std_inf | trip_gwe_std_01 | INFANT | FALSE (or TRUE if seat allocated) | 8000000.00 | 6500000.00 |
| `product_trip_pricings` | pricing_spr_ad | trip_gwe_spr_01 | ADULT | TRUE | 31500000.00 | 28000000.00 |
| `product_trip_pricings` | pricing_spr_inf | trip_gwe_spr_01 | INFANT | FALSE (or TRUE if seat allocated) | 8000000.00 | 6500000.00 |
| `product_trip_pricings` | pricing_sum_ad | trip_gwe_sum_01 | ADULT | TRUE | 12000000.00 | 10000000.00 |
| `product_trip_pricings` | pricing_sum_inf | trip_gwe_sum_01 | INFANT | FALSE | 8000000.00 | 6500000.00 |
| `product_trip_pricings` | pricing_tlp_ad | trip_gwe_tlp_01 | ADULT | TRUE | 35000000.00 | 31000000.00 |
| `product_trip_pricings` | pricing_tlp_inf | trip_gwe_tlp_01 | INFANT | FALSE (or TRUE if seat allocated) | 8500000.00 | 7000000.00 |
| `product_trip_pricings` | pricing_eb_ad | trip_gwe_eb_01 | ADULT | TRUE | 30000000.00 | 24900000.00 |
| `product_trip_pricings` | pricing_eb_inf | trip_gwe_eb_01 | INFANT | FALSE | 7500000.00 | 6000000.00 |

#### Sample `product_pricing_components` (Itemized Breakdown for GWE Summer & Classic)

> Demonstrates what each package price tier covers ($\text{selling\_price} = \text{base\_departure\_amount} + \sum \text{included\_components}$).

| id | pricing_id | Target Tier & Variant | Component Name | amount (IDR) | is_included | Description / Rationale |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `comp_sum_01` | `pricing_sum_ad` | **GWE Summer Adult (IDR 10M)** | Biaya Keberangkatan & Land Tour | 9,350,000.00 | TRUE | Bundled international flight, 4-star hotels, coach & guided tours |
| `comp_sum_02` | `pricing_sum_ad` | **GWE Summer Adult (IDR 10M)** | Schengen Visa Fee | 500,000.00 | TRUE | Official consular visa application processing fee |
| `comp_sum_03` | `pricing_sum_ad` | **GWE Summer Adult (IDR 10M)** | Airport Shuttle & Transfer | 100,000.00 | TRUE | Dedicated airport transfer between terminal and hotel |
| `comp_sum_04` | `pricing_sum_ad` | **GWE Summer Adult (IDR 10M)** | Tour Leader & Driver Tip | 50,000.00 | TRUE | Mandatory gratuity for tour leader and local bus driver |
| `comp_sum_inf_01` | `pricing_sum_inf` | **GWE Summer Infant (IDR 6.5M)** | Infant Airline Ticket & Tax | 5,500,000.00 | TRUE | Lap infant international airline ticket & government airport taxes |
| `comp_sum_inf_02` | `pricing_sum_inf` | **GWE Summer Infant (IDR 6.5M)** | Infant Travel Insurance & Admin | 1,000,000.00 | TRUE | Comprehensive medical travel insurance and administrative handling |
| `comp_std_01` | `pricing_std_ad` | **GWE Classic Adult (IDR 28.5M)**| International Flight & Accommodation | 22,000,000.00 | TRUE | Economy return flight with Qatar Airways + 6 nights twin-share hotel |
| `comp_std_02` | `pricing_std_ad` | **GWE Classic Adult (IDR 28.5M)**| Schengen Visa Fee & Assistance | 2,500,000.00 | TRUE | Full Schengen Visa consular processing and appointment handling |
| `comp_std_03` | `pricing_std_ad` | **GWE Classic Adult (IDR 28.5M)**| Airport Shuttle & Private Coach | 2,500,000.00 | TRUE | Private luxury coach for all inter-city transfers |
| `comp_std_04` | `pricing_std_ad` | **GWE Classic Adult (IDR 28.5M)**| Tour Leader & Driver Tipping | 1,500,000.00 | TRUE | Full tour duration tipping for Indonesian Tour Leader & European driver |

**Sample Add-ons (`product_addons`) — Variant Default vs Trip Override / Exclusive:**

| id | variant_id | trip_id | code | name | addon_type | charge_type | price | applicable_age_band | is_mandatory | notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `addon_gwe_01` | `var_gwe_std_26` | NULL | `ADDON-SINGLE-SUPP` | Single Supplement (Kamar Sendiri) | `SINGLE_ROOM` | `PER_ROOM` | 8500000.00 | `ADULT` | `FALSE` | **VARIANT DEFAULT:** Baseline single room rate |
| `addon_gwe_02` | `var_gwe_std_26` | NULL | `ADDON-TITLIS-ICEFLYER` | Mount Titlis Rotair Cable Car & Ice Flyer Experience | `EXPERIENTIAL_TOUR` | `PER_PAX` | 2400000.00 | `NULL` (ALL) | `FALSE` | **VARIANT DEFAULT:** Year-round alpine excursion |
| `addon_gwe_03` | `var_gwe_std_26` | NULL | `ADDON-EIFFEL-SUMMIT` | Eiffel Tower Top Summit Elevator Access | `EXPERIENTIAL_TOUR` | `PER_PAX` | 850000.00 | `NULL` (ALL) | `FALSE` | **VARIANT DEFAULT:** Paris summit access |
| `addon_gwe_04` | `var_gwe_std_26` | NULL | `ADDON-SCHENGEN-VIP` | Schengen Visa Express Consular Appointment Assistance | `VISA_EXPRESS` | `PER_PAX` | 2500000.00 | `NULL` (ALL) | `FALSE` | **VARIANT DEFAULT:** Expedited visa filing |
| `addon_tlp_ovr_01` | `var_gwe_tlp_26` | `trip_gwe_tlp_01` | `ADDON-SINGLE-SUPP` | Single Supplement (Kamar Sendiri - Peak Hotel Surcharge) | `SINGLE_ROOM` | `PER_ROOM` | 11500000.00 | `ADULT` | `FALSE` | **TRIP OVERRIDE:** Peak season hotel surcharge for Tulip Festival departure |
| `addon_tlp_exc_02` | `var_gwe_tlp_26` | `trip_gwe_tlp_01` | `ADDON-KEUKENHOF-VIP` | Keukenhof Flower Parade VIP Grandstand Access | `EXPERIENTIAL_TOUR` | `PER_PAX` | 1500000.00 | `NULL` (ALL) | `FALSE` | **TRIP EXCLUSIVE:** Special reserved grandstand seat for Bloemencorso Bollenstreek |

---

### 3. Itinerary (Variant Level Default with Trip Override)

```mermaid
erDiagram
    product_variants    ||--o{ product_itineraries     : "variant_id (default master)"
    product_trips       ||--o| product_itineraries     : "trip_id (optional override)"
    product_itineraries ||--o{ product_itinerary_items : "itinerary_id"

    product_itineraries {
        uuid      id             PK
        uuid      variant_id     FK
        uuid      trip_id        FK "NULL = variant default"
        varchar   source_type    "MERCHANT | INTERNAL"
        varchar   itinerary_type "STANDARD | CUSTOM"
        varchar   title
        text      summary
    }

    product_itinerary_items {
        uuid      id              PK
        uuid      itinerary_id    FK
        int       day_number
        int       sequence_number
        varchar   item_type       "ACTIVITY | TRANSPORT | MEAL | ACCOMMODATION | OTHER"
        varchar   title
        text      description
        uuid      poi_area_id     FK "logical FK → areas.id (POI landmark)"
        varchar   meals_included
        varchar   accommodation
    }
```

| Table | id | variant_id | trip_id | itinerary_type | title |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `product_itineraries` | itin_var_std_01 | var_gwe_std_26 | NULL | STANDARD | 7D/6N Western Europe Classic Program (Amsterdam, Paris, Swiss Alps) |
| `product_itineraries` | itin_trip_tlp_ovr | var_gwe_tlp_26 | trip_gwe_tlp_01 | CUSTOM | 7D/6N Tulip Special Keukenhof Peak Itinerary |

| Table | id | itinerary_id | day_number | sequence_number | item_type | title | poi_area_id |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `product_itinerary_items` | item_001 | itin_var_std_01 | 1 | 1 | TRANSPORT | Flight Jakarta to Amsterdam Schiphol | NULL |
| `product_itinerary_items` | item_002 | itin_var_std_01 | 2 | 1 | ACTIVITY | Keukenhof Tulip Gardens & Zaanse Schans | area_poi_keukenhof |
| `product_itinerary_items` | item_003 | itin_var_std_01 | 3 | 1 | ACTIVITY | Brussels Grand Place & Atomium Photo Stop | area_poi_atomium |
| `product_itinerary_items` | item_004 | itin_var_std_01 | 4 | 1 | ACTIVITY | Paris Highlights & Eiffel Tower Observation | area_poi_eiffel |
| `product_itinerary_items` | item_005 | itin_var_std_01 | 5 | 1 | ACTIVITY | Mount Titlis Rotair Cable Car & Glacier Excursion | area_poi_titlis |
| `product_itinerary_items` | item_006 | itin_var_std_01 | 6 | 1 | ACTIVITY | Lucerne Chapel Bridge & Zurich Old Town Leisure | area_poi_chapel_bridge |
| `product_itinerary_items` | item_007 | itin_var_std_01 | 7 | 1 | OTHER | Zurich Airport Check-in & Return Flight to Jakarta | area_poi_zurich |

---

### 4. Locations & 4-Tier Area Geography

```mermaid
erDiagram
    areas {
        uuid      id             PK "Area Domain (any tier: POI, Country, Sub-Continent, Continent)"
        uuid      parent_id      FK "Continent -> Sub Continent -> Country -> POI"
        int       area_type_id   FK
        varchar   name           "e.g. Eiffel Tower, Netherlands, Western Europe"
        varchar   code
    }

    product_locations {
        uuid      id          PK
        uuid      product_id  FK
        varchar   source_type "AREA | MANUAL"
        uuid      area_id     FK "logical FK → areas.id (POI, COUNTRY, SUB_CONTINENT, or CONTINENT)"
        varchar   area_name   "denormalized landmark/region name"
        text      address
        int       sort_order
    }

    products ||--o{ product_locations : "product_id"
    areas    ||--o{ product_locations : "area_id (Flexible anchor: POI, COUNTRY, SUB_CONTINENT, or CONTINENT)"
```

| Table | id | product_id | source_type | area_id | area_name | Anchored Level | Resolved Upward Flat Hierarchy | sort_order |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `product_locations` | loc_01 | prod_gwe_01 | AREA | 550e8400-e29b-41d4-a716-446655440001 | Keukenhof Gardens | **POI** (Tier 4) | `continent: Europe, subContinent: Western Europe, country: Netherlands, poi: Keukenhof Gardens` | 1 |
| `product_locations` | loc_02 | prod_gwe_01 | AREA | 550e8400-e29b-41d4-a716-446655440002 | Eiffel Tower Paris | **POI** (Tier 4) | `continent: Europe, subContinent: Western Europe, country: France, poi: Eiffel Tower Paris` | 2 |
| `product_locations` | loc_03 | prod_jp_01 | AREA | 550e8400-e29b-41d4-a716-446655440010 | Japan | **COUNTRY** (Tier 3) | `continent: Asia, subContinent: East Asia, country: Japan, poi: NULL` | 1 |
| `product_locations` | loc_04 | prod_nordic_01 | AREA | 550e8400-e29b-41d4-a716-446655440020 | Scandinavia & Nordics | **SUB_CONTINENT** (Tier 2) | `continent: Europe, subContinent: Northern Europe, country: NULL, poi: NULL` | 1 |
| `product_locations` | loc_05 | prod_safari_01 | AREA | 550e8400-e29b-41d4-a716-446655440030 | Africa | **CONTINENT** (Tier 1) | `continent: Africa, subContinent: NULL, country: NULL, poi: NULL` | 1 |

---

### 5. Media

```mermaid
erDiagram
    products {
        uuid id PK
    }

    product_media {
        uuid      id               PK
        uuid      product_id       FK
        varchar   storage_provider "DATABASE | S3 | CLOUDFLARE_R2"
        varchar   media_type       "IMAGE | VIDEO"
        varchar   file_name        "original filename"
        bigint    file_size_bytes  "bytes"
        varchar   mime_type        "image/jpeg, video/mp4, etc."
        varchar   object_key       "nullable in Phase 1"
        varchar   url              "stream or CDN URL"
        timestamp created_at
        timestamp updated_at
    }

    product_media_blobs {
        uuid      media_id         PK "FK to product_media.id"
        bytea     file_data        "binary data (Phase 1)"
        timestamp created_at
    }

    product_media_usages {
        uuid      id            PK
        uuid      media_id      FK
        varchar   target_type   "PRODUCT | VARIANT | ITINERARY_ITEM"
        uuid      target_id     "polymorphic"
        varchar   usage_context "COVER | GALLERY | THUMBNAIL | ATTACHMENT"
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products            ||--o{ product_media       : "product_id"
    product_media       ||--o| product_media_blobs : "binary data (Phase 1)"
    product_media       ||--o{ product_media_usages: "media_id"
```

| Table           | id        | product_id     | storage_provider | media_type | file_name                             | file_size_bytes | mime_type       | object_key                                        | url                                                              |
| --------------- | --------- | -------------- | ---------------- | ---------- | ------------------------------------- | --------------- | --------------- | ------------------------------------------------- | ---------------------------------------------------------------- |
| `product_media` | media_001 | prod_gwe_01    | DATABASE         | IMAGE      | gwe-eiffel-tower-hero.jpg             | 1845200         | image/jpeg      | NULL                                              | /api/v1/media/media_001/stream                                   |
| `product_media` | media_002 | prod_gwe_01    | DATABASE         | IMAGE      | gwe-titlis-glacier-panorama.jpg       | 2450100         | image/jpeg      | NULL                                              | /api/v1/media/media_002/stream                                   |
| `product_media` | media_003 | prod_gwe_01    | DATABASE         | IMAGE      | gwe-keukenhof-tulips.jpg              | 2100400         | image/jpeg      | NULL                                              | /api/v1/media/media_003/stream                                   |

| Table                  | id        | media_id  | target_type | target_id      | usage_context | sort_order |
| ---------------------- | --------- | --------- | ----------- | -------------- | ------------- | ---------- |
| `product_media_usages` | usage_001 | media_001 | PRODUCT     | prod_gwe_01    | COVER         | 1          |
| `product_media_usages` | usage_002 | media_002 | PRODUCT     | prod_gwe_01    | GALLERY       | 1          |
| `product_media_usages` | usage_003 | media_003 | PRODUCT     | prod_gwe_01    | GALLERY       | 2          |

---

### 6. Supplementary Content

```mermaid
erDiagram
    products {
        uuid id PK
    }

    product_supplementaries {
        uuid      id          PK
        uuid      product_id  FK
        varchar   target_type "PRODUCT | VARIANT | TRIP"
        uuid      target_id   "polymorphic"
        varchar   category    "INCLUDED | EXCLUDED | IMPORTANT_INFO | NOTE"
        text      content
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products ||--o{ product_supplementaries : "product_id"
```

| Table                     | id       | product_id     | target_type | target_id      | category       | content                                                                                                                                                                                  | sort_order |
| ------------------------- | -------- | -------------- | ----------- | -------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `product_supplementaries` | supp_001 | prod_gwe_01    | PRODUCT     | prod_gwe_01    | IMPORTANT_INFO | Valid Schengen Visa and Indonesian passport with at least 6 months validity required from travel date.                                                                                   | 1          |
| `product_supplementaries` | supp_002 | prod_gwe_01    | PRODUCT     | prod_gwe_01    | INCLUDED       | International flight tickets (economy class), 4-star hotel accommodations (twin-share), private luxury coach transfers, daily breakfast & halal/Muslim-friendly meals, and tour leader. | 2          |

---

### 7. High-Level Domain Overview

```mermaid
flowchart LR
    subgraph TAXONOMY["🏷️ Multi-Dimensional Taxonomy"]
        DIM["category_dimensions\n(Travel Style, Theme, Season, etc.)"]
        CAT["product_categories\n(Taxonomy Nodes)"]
        PCA["product_category_assignments\n(Modular Scoped M:N)"]
    end

    subgraph CORE["🏷️ L1 — Core Umbrella"]
        P["products\n(Master Tour)"]
        PJ["product_journeys\n(Base Duration)"]
    end

    subgraph HIERARCHY["🗂️ L2/L3 — Hierarchy & Pricing"]
        PV["product_variants\n(Listing Card)"]
        PT["product_trips\n(Dated Departure)"]
        PP["product_trip_pricings\n(Age Bands & Quota)"]
        PPC["product_pricing_components\n(Itemized Breakdown)"]
        ADD["product_addons\n(Optional Extras)"]
        BDG["product_badges\n(Pill Labels)"]
        PVB["product_variant_badges\n(M:N)"]
    end

    subgraph CONTENT["📝 Content"]
        ITN["product_itineraries\n(Variant Default / Trip Override)"]
        ITEM["product_itinerary_items"]
        LOC["product_locations"]
        SUPP["product_supplementaries"]
    end

    subgraph MEDIA["🖼️ Media"]
        M["product_media"]
        MB["product_media_blobs\n(Phase 1 Binary)"]
        MU["product_media_usages\n(Polymorphic)"]
    end

    subgraph SEO["🔍 SEO & Social"]
        SEO_M["seo_metadata\n(Polymorphic)"]
    end

    DIM -->|"1:N"| CAT
    CAT -->|"1:N"| PCA
    PCA -->|"Product-Level (L1)"| P
    PCA -->|"Variant-Level (L2)"| PV

    P   -->|"1:1"| PJ
    P   -->|"1:N"| PV
    PV  -->|"1:N"| PT
    PT  -->|"1:N"| PP
    PP  -->|"1:N"| PPC
    PV  -->|"1:N"| ADD
    PV  -->|"1:N"| PVB
    BDG -->|"1:N"| PVB

    PV  -->|"Default (trip_id IS NULL)"| ITN
    PT  -->|"Override (trip_id IS NOT NULL)"| ITN
    ITN -->|"1:N"| ITEM
    P   -->|"1:N"| LOC
    P   -->|"1:N"| SUPP

    P   -->|"1:N"| M
    M   -->|"1:1"| MB
    M   -->|"1:N"| MU

    MU  -. "PRODUCT" .-> P
    MU  -. "VARIANT" .-> PV
    MU  -. "ITINERARY_ITEM" .-> ITEM

    SUPP -. "VARIANT" .-> PV
    SUPP -. "TRIP" .-> PT

    SEO_M -. "PRODUCT" .-> P
    SEO_M -. "VARIANT" .-> PV
```

---

## 📐 Index Summary

| Index Name                              | Table                          | Columns                                                                                          | Type                  | Purpose                                             |
| --------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------ | --------------------- | --------------------------------------------------- |
| `idx_dimensions_code`                   | `category_dimensions`          | `(code)`                                                                                         | B-Tree unique         | Dimension code lookup                               |
| `idx_categories_dimension_id`           | `product_categories`           | `(dimension_id)`                                                                                 | B-Tree                | Filter categories by taxonomy dimension             |
| `idx_categories_parent_id`              | `product_categories`           | `(parent_id)`                                                                                    | B-Tree                | Parent-child taxonomy traversal                     |
| `idx_categories_slug`                   | `product_categories`           | `(slug)`                                                                                         | B-Tree                | Category lookup by slug                             |
| `idx_cat_assign_product_id`             | `product_category_assignments` | `(product_id)`                                                                                   | B-Tree                | Fetch all categories assigned to product (L1)       |
| `idx_cat_assign_variant_id`             | `product_category_assignments` | `(variant_id)`                                                                                   | B-Tree                | Fetch variant-specific category overrides (L2)      |
| `idx_cat_assign_category_id`            | `product_category_assignments` | `(category_id)`                                                                                  | B-Tree                | Reverse lookup products/variants by category        |
| `idx_cat_assign_lookup`                 | `product_category_assignments` | `(category_id, product_id, variant_id)`                                                          | B-Tree composite      | Fast category filter evaluation in search queries   |
| `idx_badges_code`                       | `product_badges`               | `(code)`                                                                                         | B-Tree unique         | Promotional badge code lookup                       |
| `idx_var_badges_variant_id`             | `product_variant_badges`       | `(variant_id)`                                                                                   | B-Tree                | Fetch badges for variant card                       |
| `idx_var_badges_badge_id`               | `product_variant_badges`       | `(badge_id)`                                                                                     | B-Tree                | Reverse lookup variants by badge                    |
| `idx_var_badges_lookup`                 | `product_variant_badges`       | `(badge_id, variant_id)`                                                                         | B-Tree composite      | Fast promotional badge filter evaluation in search  |
| `idx_products_status`                   | `products`                     | `(listing_status)` WHERE `deleted_at IS NULL`                                                    | B-Tree partial        | Active product listing                              |
| `idx_products_name_trgm`                | `products`                     | `(name)`                                                                                         | GIN pg_trgm           | Search by product name                              |
| `idx_products_slug_trgm`                | `products`                     | `(slug)`                                                                                         | GIN pg_trgm           | Destination text search                             |
| `idx_variants_product_id`               | `product_variants`             | `(product_id)`                                                                                   | B-Tree                | Variant lookup by product                           |
| `idx_variants_name_trgm`                | `product_variants`             | `(name)`                                                                                         | GIN pg_trgm           | Variant name text search                            |
| `idx_variants_slug_trgm`                | `product_variants`             | `(slug)`                                                                                         | GIN pg_trgm           | Variant slug text search                            |
| `idx_trips_variant_id`                  | `product_trips`                | `(variant_id)`                                                                                   | B-Tree                | Trip lookup by variant                              |
| `idx_trips_search`                      | `product_trips`                | `(start_date, min_quota, max_quota)` WHERE `status = 'ACTIVE'`                                   | B-Tree partial        | Search date+total pack (quota) filter               |
| `idx_pricings_search`                   | `product_trip_pricings`        | `(trip_id, age_band, selling_price)`                                                             | B-Tree                | Price range filter and starting price lookup        |
| `idx_pricing_components_pricing_id`     | `product_pricing_components`   | `(pricing_id)`                                                                                   | B-Tree                | Itemized pricing component lookup by pricing tier   |
| `idx_addons_variant_trip`               | `product_addons`               | `(variant_id, trip_id)`                                                                          | B-Tree                | Add-on lookup by variant and trip                   |
| `uq_addon_variant_code`                 | `product_addons`               | `(variant_id, code)` WHERE `trip_id IS NULL`                                                     | B-Tree unique partial | Enforce 1 default add-on per code per variant       |
| `uq_addon_trip_code`                    | `product_addons`               | `(trip_id, code)` WHERE `trip_id IS NOT NULL`                                                    | B-Tree unique partial | Enforce 1 override/exclusive add-on per code per trip|
| `uq_itinerary_variant_default`          | `product_itineraries`          | `(variant_id)` WHERE `trip_id IS NULL`                                                           | B-Tree unique partial | Enforce 1 master default itinerary per variant      |
| `uq_itinerary_trip_override`            | `product_itineraries`          | `(trip_id)` WHERE `trip_id IS NOT NULL`                                                          | B-Tree unique partial | Enforce 1 custom override itinerary per trip        |
| `idx_itinerary_items_itinerary_id`      | `product_itinerary_items`      | `(itinerary_id)`                                                                                 | B-Tree                | Itinerary item lookup by itinerary                  |
| `idx_itinerary_items_poi`               | `product_itinerary_items`      | `(poi_area_id)`                                                                                  | B-Tree                | Itinerary item POI landmark join                    |
| `idx_locations_product_id`              | `product_locations`            | `(product_id)`                                                                                   | B-Tree                | Location lookup by product                          |
| `idx_locations_area_id`                 | `product_locations`            | `(area_id)`                                                                                      | B-Tree                | Join to Area hierarchy (anchored to POI/Country...) |
| `idx_locations_area_name_trgm`          | `product_locations`            | `(area_name)`                                                                                    | GIN pg_trgm           | Destination text search                             |
| `idx_media_product_id`                  | `product_media`                | `(product_id)`                                                                                   | B-Tree                | Media lookup by product                             |
| `idx_media_type`                        | `product_media`                | `(media_type)`                                                                                   | B-Tree                | Filter media by visual type (IMAGE / VIDEO)         |
| `idx_media_storage`                     | `product_media`                | `(storage_provider)`                                                                             | B-Tree                | Storage provider lookup (DATABASE / S3 / R2)        |
| `idx_media_usages_target`               | `product_media_usages`         | `(target_type, target_id)`                                                                       | B-Tree                | Polymorphic media lookup                            |
| `uq_media_usages_single_cover`          | `product_media_usages`         | `(target_id, usage_context)` WHERE `usage_context = 'COVER'`                                     | B-Tree unique partial | Enforce single COVER image per entity target        |
| `idx_supplementaries_target`            | `product_supplementaries`      | `(target_type, target_id)`                                                                       | B-Tree                | Polymorphic content lookup                          |
| `idx_seo_target`                        | `seo_metadata`                 | `(target_type, target_id)`                                                                       | B-Tree                | Polymorphic SEO metadata lookup                     |


