# Product Domain — Technical Data Model & Architecture

> **Overview**
> Single source of truth for the Product Domain PostgreSQL schema. This document covers the complete DDL for all product tables, architectural principles, and per-table ERDs with sample data.
>
> _Engineered for High Scalability, Data Integrity, and optimized for a NestJS + PostgreSQL stack._

**Document Map:**

| Document | Responsibility |
|---|---|
| **This file** | Full DDL schema, per-table ERDs, sample data, architecture principles |
| [Product Hierarchy](./product-hierarchy-technical-design.md) | 3-level hierarchy mental model, full-domain ERD, hierarchy sample data (GWE) |
| [Search & Filter](./product-search-filter-technical-design.md) | Search API contract, SQL query, indexing strategy |

---

## 🏗️ Architecture & Engineering Principles

### 1. Client-Optimized Data Aggregation
Backend must serve aggregated JSON payloads — never raw rows. Avoid N+1 queries by joining variants, trips, pricing, and media in a **single database round-trip** using TypeORM query builders or raw SQL with `json_agg`.

### 2. Schema Version Control
All DDL scripts are the source of truth for ORM migrations. Every schema change must be committed as a versioned migration file and run through CI/CD pipelines — never applied manually in production.

### 3. Polymorphic Relationships
Media usages, supplementary content, and FAQs use `(target_type, target_id)` to target multiple entity types from one table. Rules:

- **DB-level:** composite B-Tree index on `(target_type, target_id)` is mandatory for read performance.
- **App-level:** NestJS service layer is responsible for cascading deletes to polymorphic child rows inside a **database transaction** — no database FK can enforce this.
- **Valid `target_type` values:** `PRODUCT` · `VARIANT` · `TRIP` · `ITINERARY_ITEM`

### 4. Concurrency & Quota Management
`product_trips.max_quota` is an **immutable ceiling** set at product configuration time. During booking, use **Pessimistic Locking** (`SELECT ... FOR UPDATE`) on the trip row to prevent race conditions. Live availability is tracked in a separate `product_trip_bookings` table — never mutate `max_quota`.

### 5. Precision Economics
All price columns use `DECIMAL(15,2)`. Never use `FLOAT` or `DOUBLE` for monetary values. Use `decimal.js` or `big.js` in NestJS for all arithmetic before returning to clients.

### 6. Media & Document (PDF) Lifecycle & Validation
`product_media` serves as the centralized repository for all binary assets: images, videos, and PDF documents.
- **MIME & Byte Validation:** PDF uploads require strict backend validation for `application/pdf` along with magic byte (`%PDF-`) verification in the NestJS upload interceptor. Maximum file size is strictly capped (e.g., 20 MB).
- **Single Itinerary PDF Guarantee:** A partial unique index (`uq_media_usages_product_itinerary_pdf`) on `product_media_usages` guarantees at the database level that a product can have at most **one** active `ITINERARY_PDF` usage.
- **Client Download Disposition:** Media records store `file_name` and `file_size_bytes` so UI clients can display human-readable labels and file size indicators (`PDF • 4.6 MB`). CDN / S3 headers must stream downloads with `Content-Disposition: attachment; filename="<file_name>"`.
- **Fast Read Redundancy:** While `product_media` is the authoritative source of file metadata, `products.itinerary_pdf_url` stores the denormalized active CDN URL for instant O(1) reads without table joins.

### 7. Audit Timestamps & State Traceability (`created_at`, `updated_at`, `deleted_at`)
Every domain table strictly maintains timestamp tracking for auditability, cache invalidation, and data synchronization:
- **`created_at` & `updated_at`:** Every table enforces `TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP`. A database trigger function (`set_updated_at_timestamp()`) automatically updates `updated_at` on row mutation to prevent stale data even during direct SQL operations.
- **Soft Deletes (`deleted_at`):** High-value catalog entities (`products`, `product_variants`) use nullable `deleted_at` timestamps instead of physical deletion. Partial indexes explicitly exclude soft-deleted rows (`WHERE deleted_at IS NULL`) to maintain query performance.

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
-- 1. TAXONOMY
-- =========================================================================
CREATE TABLE product_categories (
    id          BIGSERIAL    PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);


-- =========================================================================
-- 2. CORE — L1: products & product_journeys
-- =========================================================================

-- Master product entity — the brand/program umbrella
CREATE TABLE products (
    id                UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    category_id       BIGINT       REFERENCES product_categories(id) ON DELETE RESTRICT,
    product_type      VARCHAR(50)  NOT NULL,                    -- e.g. JOURNEY
    code              VARCHAR(100) UNIQUE NOT NULL,             -- e.g. GWE
    slug              VARCHAR(255) UNIQUE NOT NULL,             -- e.g. grand-west-europe
    itinerary_pdf_url VARCHAR(500),                             -- 1 Product : 1 PDF Itinerary file (shared across variants)
    listing_status    VARCHAR(50)  NOT NULL DEFAULT 'DRAFT',    -- DRAFT | ACTIVE | ARCHIVED
    created_at        TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at        TIMESTAMP    NULL
);
CREATE INDEX idx_products_category    ON products(category_id);
CREATE INDEX idx_products_status      ON products(listing_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_slug_trgm   ON products USING GIN (slug gin_trgm_ops);

-- Base journey metadata (1:1 with products)
CREATE TABLE product_journeys (
    product_id          UUID        PRIMARY KEY REFERENCES products(id) ON DELETE CASCADE,
    nationality_scope   VARCHAR(50) NOT NULL DEFAULT 'ALL',
    duration_days       INT         NOT NULL DEFAULT 1,
    duration_nights     INT         NOT NULL DEFAULT 0,
    created_at          TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_duration CHECK (duration_days >= 1 AND duration_nights >= 0)
);


-- =========================================================================
-- 3. HIERARCHY — L2: product_variants
-- One product → many variants. Each variant is one card on All Tours.
-- =========================================================================
CREATE TABLE product_variants (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id      UUID        NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,             -- e.g. "GWE Spring 2026"
    slug            VARCHAR(255) UNIQUE NOT NULL,       -- e.g. "gwe-spring-2026"
    code            VARCHAR(100) UNIQUE NOT NULL,       -- e.g. "GWE-SPR-2026"

    -- Duration override: NULL means inherit from product_journeys via COALESCE
    duration_days   INT         NULL,
    duration_nights INT         NULL,

    listing_status  VARCHAR(50) NOT NULL DEFAULT 'DRAFT',  -- DRAFT | ACTIVE | ARCHIVED
    created_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at      TIMESTAMP   NULL
);
CREATE INDEX idx_variants_product_id   ON product_variants(product_id);
CREATE INDEX idx_variants_slug         ON product_variants(slug);
CREATE INDEX idx_variants_status       ON product_variants(listing_status) WHERE deleted_at IS NULL;
CREATE INDEX idx_variants_name_trgm    ON product_variants USING GIN (name gin_trgm_ops);


-- =========================================================================
-- 4. HIERARCHY — L3: product_trips & product_trip_pricings
-- One variant → many trips. A trip is a concrete dated departure window.
-- =========================================================================
CREATE TABLE product_trips (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    variant_id      UUID        NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    start_date      DATE        NOT NULL,
    end_date        DATE        NOT NULL,
    min_quota       INT         NOT NULL DEFAULT 1,
    max_quota       INT         NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    -- ACTIVE | FULL | CANCELLED | COMPLETED
    created_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_trip_variant_start UNIQUE (variant_id, start_date),
    CONSTRAINT chk_trip_dates        CHECK  (end_date > start_date),
    CONSTRAINT chk_trip_quota        CHECK  (max_quota >= min_quota AND min_quota >= 1)
);
CREATE INDEX idx_trips_variant_id ON product_trips(variant_id);
-- Partial index: search queries only touch ACTIVE trips
CREATE INDEX idx_trips_search     ON product_trips(start_date, max_quota)
    WHERE status = 'ACTIVE';

-- Pricing tiers per trip (e.g. per nationality scope)
CREATE TABLE product_trip_pricings (
    id                  UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    trip_id             UUID           NOT NULL REFERENCES product_trips(id) ON DELETE CASCADE,
    nationality_scope   VARCHAR(50)    NOT NULL DEFAULT 'ALL',
    base_price          DECIMAL(15,2)  NOT NULL,
    selling_price       DECIMAL(15,2)  NOT NULL,
    created_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_pricing_trip_scope UNIQUE (trip_id, nationality_scope),
    CONSTRAINT chk_price_sanity      CHECK  (selling_price > 0 AND base_price >= selling_price)
);


-- =========================================================================
-- 5. CONTENT — Itinerary
-- Day-by-day programme. Owned at the product level (shared across variants).
-- =========================================================================
CREATE TABLE product_itineraries (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id      UUID        NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    source_type     VARCHAR(50) NOT NULL,       -- MERCHANT | INTERNAL
    itinerary_type  VARCHAR(50) NOT NULL,       -- STANDARD | CUSTOM
    created_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE product_itinerary_items (
    id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    itinerary_id    UUID        NOT NULL REFERENCES product_itineraries(id) ON DELETE CASCADE,
    day_number      INT         NOT NULL,
    sequence_number INT         NOT NULL,
    item_type       VARCHAR(50) NOT NULL,       -- ACTIVITY | TRANSPORT | MEAL | ACCOMMODATION
    title           VARCHAR(255),
    description     TEXT,
    created_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_itinerary_item_order UNIQUE (itinerary_id, day_number, sequence_number)
);


-- =========================================================================
-- 6. CONTENT — Locations
-- Multiple destination markers per product. area_id is a logical FK to the
-- Area/Geography domain (not enforced at DB level across domains).
-- =========================================================================
CREATE TABLE product_locations (
    id          UUID           PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID           NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    source_type VARCHAR(50)    NOT NULL,                   -- AREA | MANUAL
    area_id     UUID           NOT NULL,                   -- Logical FK → Area domain
    area_name   VARCHAR(100),                              -- Denormalized for fast UI rendering
    lat         DOUBLE PRECISION,
    lng         DOUBLE PRECISION,
    address     TEXT,
    sort_order  INT            NOT NULL DEFAULT 0,
    created_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP      NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_locations_product_id    ON product_locations(product_id);
CREATE INDEX idx_locations_area_name_trgm ON product_locations USING GIN (area_name gin_trgm_ops);


-- =========================================================================
-- 7. MEDIA — product_media & product_media_usages (polymorphic)
-- Media assets are owned at the product level.
-- Usage slots (cover, gallery, thumbnail, itinerary_pdf) target entities polymorphically.
-- Supports images, videos, and document assets (PDF brochures).
-- =========================================================================
CREATE TABLE product_media (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id       UUID         NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    source_upload_id VARCHAR(255),                          -- CDN / upload service ref
    media_type       VARCHAR(50)  NOT NULL,                  -- IMAGE | VIDEO | PDF
    file_name        VARCHAR(255),                          -- Original / download filename (e.g. Turkey-Wonders-Itinerary.pdf)
    file_size_bytes  BIGINT,                                -- File size in bytes for UI preview
    mime_type        VARCHAR(100),                          -- e.g. application/pdf, image/jpeg, video/mp4
    object_key       VARCHAR(500) NOT NULL,                 -- S3/GCS object key
    url              VARCHAR(500) NOT NULL,                 -- CDN public URL
    created_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_media_type CHECK (media_type IN ('IMAGE', 'VIDEO', 'PDF'))
);

CREATE TABLE product_media_usages (
    id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    media_id      UUID        NOT NULL REFERENCES product_media(id) ON DELETE CASCADE,
    target_type   VARCHAR(50) NOT NULL,    -- PRODUCT | VARIANT | ITINERARY_ITEM
    target_id     UUID        NOT NULL,    -- Polymorphic — resolved by target_type
    usage_context VARCHAR(50) NOT NULL,    -- COVER | GALLERY | THUMBNAIL | ITINERARY_PDF | ATTACHMENT
    sort_order    INT         NOT NULL DEFAULT 0,
    created_at    TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_media_target_type    CHECK (target_type IN ('PRODUCT', 'VARIANT', 'ITINERARY_ITEM')),
    CONSTRAINT chk_media_usage_context CHECK (usage_context IN ('COVER', 'GALLERY', 'THUMBNAIL', 'ITINERARY_PDF', 'ATTACHMENT'))
);
-- Essential composite index for polymorphic reads
CREATE INDEX idx_media_usages_target ON product_media_usages(target_type, target_id);

-- Enforce strict 1:1 rule: exactly 1 active ITINERARY_PDF per Product
CREATE UNIQUE INDEX uq_media_usages_product_itinerary_pdf
    ON product_media_usages(target_id, usage_context)
    WHERE target_type = 'PRODUCT' AND usage_context = 'ITINERARY_PDF';


-- =========================================================================
-- 8. SUPPLEMENTARY CONTENT (polymorphic)
-- Reusable content blocks and FAQs that can target any entity level.
-- =========================================================================
CREATE TABLE product_supplementaries (
    id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID        NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    target_type VARCHAR(50) NOT NULL,    -- PRODUCT | VARIANT | TRIP
    target_id   UUID        NOT NULL,    -- Polymorphic
    category    VARCHAR(50) NOT NULL,    -- INCLUDED | EXCLUDED | IMPORTANT_INFO | NOTE
    content     TEXT,
    sort_order  INT         NOT NULL DEFAULT 0,
    created_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_supplementaries_target ON product_supplementaries(target_type, target_id);

CREATE TABLE product_faqs (
    id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id  UUID        NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    target_type VARCHAR(50) NOT NULL,    -- PRODUCT | VARIANT | TRIP
    target_id   UUID        NOT NULL,    -- Polymorphic
    question    TEXT        NOT NULL,
    answer      TEXT,
    sort_order  INT         NOT NULL DEFAULT 0,
    created_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_faqs_target ON product_faqs(target_type, target_id);


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
CREATE TRIGGER trg_products_updated_at             BEFORE UPDATE ON products             FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_categories_updated_at   BEFORE UPDATE ON product_categories   FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_journeys_updated_at     BEFORE UPDATE ON product_journeys     FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_variants_updated_at     BEFORE UPDATE ON product_variants     FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_trips_updated_at        BEFORE UPDATE ON product_trips        FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_trip_pricings_updated_at BEFORE UPDATE ON product_trip_pricings FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_itineraries_updated_at  BEFORE UPDATE ON product_itineraries  FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_itinerary_items_updated_at BEFORE UPDATE ON product_itinerary_items FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_locations_updated_at    BEFORE UPDATE ON product_locations    FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_media_updated_at        BEFORE UPDATE ON product_media        FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_media_usages_updated_at BEFORE UPDATE ON product_media_usages FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_supplementaries_updated_at BEFORE UPDATE ON product_supplementaries FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_product_faqs_updated_at         BEFORE UPDATE ON product_faqs         FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
```

---

## 📊 Domain Data Scenario — Turkey Wonders

**Product:** Turkey Wonders · **ID:** `prod_turkey_01`

_Each section below shows the ERD for that sub-domain, followed by concrete sample rows._

---

### 1. Core Product & Journey

```mermaid
erDiagram
    product_categories {
        bigint    id   PK
        varchar   name
        timestamp created_at
        timestamp updated_at
    }

    products {
        uuid      id              PK
        bigint    category_id     FK
        varchar   product_type
        varchar   code
        varchar   slug
        varchar   itinerary_pdf_url "1 product : 1 PDF file"
        varchar   listing_status
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    product_journeys {
        uuid      product_id         PK "FK → products"
        varchar   nationality_scope
        int       duration_days
        int       duration_nights
        timestamp created_at
        timestamp updated_at
    }

    product_categories ||--o{ products       : "category_id"
    products           ||--o| product_journeys: "product_id (1:1)"
```

| Table | id | name | created_at | updated_at |
|---|---|---|---|---|
| `product_categories` | 10 | Cultural & Heritage Tours | 2026-01-01 00:00:00 | 2026-01-01 00:00:00 |

| Table | id | category_id | product_type | code | slug | itinerary_pdf_url | listing_status | created_at | updated_at | deleted_at |
|---|---|---|---|---|---|---|---|---|---|---|
| `products` | prod_turkey_01 | 10 | JOURNEY | TURKEY-WONDERS | turkey-wonders | https://cdn.hobiholidays.com/docs/itineraries/turkey-wonders.pdf | ACTIVE | 2026-01-10 09:00:00 | 2026-01-10 09:00:00 | NULL |

| Table | product_id | nationality_scope | duration_days | duration_nights | created_at | updated_at |
|---|---|---|---|---|---|---|
| `product_journeys` | prod_turkey_01 | ALL | 9 | 8 | 2026-01-10 09:00:00 | 2026-01-10 09:00:00 |

---

### 2. Variants, Trips & Pricing

> Full hierarchy ERDs and GWE sample data: [Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)

```mermaid
erDiagram
    products {
        uuid id PK
    }

    product_variants {
        uuid      id             PK
        uuid      product_id     FK
        varchar   name
        varchar   slug
        varchar   code
        int       duration_days  "NULL = inherit"
        int       duration_nights
        varchar   listing_status
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    product_trips {
        uuid      id         PK
        uuid      variant_id FK
        date      start_date
        date      end_date
        int       min_quota
        int       max_quota
        varchar   status
        timestamp created_at
        timestamp updated_at
    }

    product_trip_pricings {
        uuid       id                 PK
        uuid       trip_id            FK
        varchar    nationality_scope
        decimal    base_price
        decimal    selling_price
        timestamp  created_at
        timestamp  updated_at
    }

    products          ||--o{ product_variants    : "product_id"
    product_variants  ||--o{ product_trips       : "variant_id"
    product_trips     ||--o{ product_trip_pricings: "trip_id"
```

| Table | id | product_id | name | slug | code | duration_days | duration_nights | listing_status | created_at | updated_at | deleted_at |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `product_variants` | var_turkey_oct | prod_turkey_01 | Turkey Wonders Oct 2026 | turkey-wonders-oct-2026 | TURKEY-OCT-2026 | NULL | NULL | ACTIVE | 2026-01-10 10:00:00 | 2026-01-10 10:00:00 | NULL |

| Table | id | variant_id | start_date | end_date | min_quota | max_quota | status | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|
| `product_trips` | trip_tur_001 | var_turkey_oct | 2026-10-10 | 2026-10-18 | 10 | 30 | ACTIVE | 2026-01-10 10:30:00 | 2026-01-10 10:30:00 |

| Table | id | trip_id | nationality_scope | base_price | selling_price | created_at | updated_at |
|---|---|---|---|---|---|---|---|
| `product_trip_pricings` | pricing_001 | trip_tur_001 | ALL | 25000000.00 | 22000000.00 | 2026-01-10 11:00:00 | 2026-01-10 11:00:00 |

---

### 3. Itinerary

```mermaid
erDiagram
    products {
        uuid id PK
    }

    product_itineraries {
        uuid      id             PK
        uuid      product_id     FK
        varchar   source_type
        varchar   itinerary_type
        timestamp created_at
        timestamp updated_at
    }

    product_itinerary_items {
        uuid      id              PK
        uuid      itinerary_id    FK
        int       day_number
        int       sequence_number
        varchar   item_type
        varchar   title
        text      description
        timestamp created_at
        timestamp updated_at
    }

    products             ||--o{ product_itineraries     : "product_id"
    product_itineraries  ||--o{ product_itinerary_items : "itinerary_id"
```

| Table | id | product_id | source_type | itinerary_type | created_at | updated_at |
|---|---|---|---|---|---|---|
| `product_itineraries` | itinerary_001 | prod_turkey_01 | MERCHANT | STANDARD | 2026-01-10 11:30:00 | 2026-01-10 11:30:00 |

| Table | id | itinerary_id | day_number | sequence_number | item_type | title | description | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|
| `product_itinerary_items` | item_001 | itinerary_001 | 1 | 1 | ACTIVITY | Istanbul City Tour | Arrive at Istanbul Airport, meet tour guide, visit Blue Mosque and Hagia Sophia. | 2026-01-10 11:45:00 | 2026-01-10 11:45:00 |
| `product_itinerary_items` | item_002 | itinerary_001 | 3 | 1 | ACTIVITY | Cappadocia Hot Air Balloon | Sunrise hot air balloon flight over Göreme Valley followed by traditional breakfast. | 2026-01-10 11:45:00 | 2026-01-10 11:45:00 |
| `product_itinerary_items` | item_003 | itinerary_001 | 6 | 1 | ACTIVITY | Bursa Grand Mosque | Explore historical Ottoman architecture and silk market in Bursa. | 2026-01-10 11:45:00 | 2026-01-10 11:45:00 |

---

### 4. Locations

```mermaid
erDiagram
    products {
        uuid id PK
    }

    product_locations {
        uuid      id          PK
        uuid      product_id  FK
        varchar   source_type
        uuid      area_id     "logical FK → Area domain"
        varchar   area_name   "denormalized"
        float     lat
        float     lng
        text      address
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products ||--o{ product_locations : "product_id"
```

| Table | id | product_id | source_type | area_id | area_name | lat | lng | address | sort_order | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `product_locations` | loc_01 | prod_turkey_01 | AREA | 550e8400-e29b-41d4-a716-446655440001 | Istanbul | 41.0082 | 28.9784 | Sultanahmet, Istanbul, Turkey | 1 | 2026-01-10 12:00:00 | 2026-01-10 12:00:00 |
| `product_locations` | loc_02 | prod_turkey_01 | AREA | 550e8400-e29b-41d4-a716-446655440002 | Cappadocia | 38.6431 | 34.8289 | Göreme, Nevşehir, Turkey | 2 | 2026-01-10 12:00:00 | 2026-01-10 12:00:00 |
| `product_locations` | loc_03 | prod_turkey_01 | AREA | 550e8400-e29b-41d4-a716-446655440003 | Bursa | 40.1885 | 29.0610 | Osmangazi, Bursa, Turkey | 3 | 2026-01-10 12:00:00 | 2026-01-10 12:00:00 |

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
        varchar   media_type       "IMAGE | VIDEO | PDF"
        varchar   file_name        "original filename"
        bigint    file_size_bytes  "bytes"
        varchar   mime_type        "application/pdf etc"
        varchar   object_key
        varchar   url
        timestamp created_at
        timestamp updated_at
    }

    product_media_usages {
        uuid      id            PK
        uuid      media_id      FK
        varchar   target_type   "PRODUCT | VARIANT | ITINERARY_ITEM"
        uuid      target_id     "polymorphic"
        varchar   usage_context "COVER | GALLERY | THUMBNAIL | ITINERARY_PDF"
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products      ||--o{ product_media       : "product_id"
    product_media ||--o{ product_media_usages: "media_id"
```

| Table | id | product_id | source_upload_id | media_type | file_name | file_size_bytes | mime_type | object_key | url | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `product_media` | media_001 | prod_turkey_01 | upl_img_001 | IMAGE | hot-air-balloon.jpg | 1845200 | image/jpeg | products/turkey/hot-air-balloon.jpg | https://cdn.hobiholidays.com/products/turkey/hot-air-balloon.jpg | 2026-01-10 12:30:00 | 2026-01-10 12:30:00 |
| `product_media` | media_002 | prod_turkey_01 | upl_doc_001 | PDF | Turkey-Wonders-Official-Itinerary.pdf | 4613734 | application/pdf | products/turkey/docs/turkey-wonders-itinerary.pdf | https://cdn.hobiholidays.com/docs/itineraries/turkey-wonders.pdf | 2026-01-10 12:35:00 | 2026-01-10 12:35:00 |

| Table | id | media_id | target_type | target_id | usage_context | sort_order | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|
| `product_media_usages` | usage_001 | media_001 | PRODUCT | prod_turkey_01 | COVER | 1 | 2026-01-10 12:40:00 | 2026-01-10 12:40:00 |
| `product_media_usages` | usage_002 | media_002 | PRODUCT | prod_turkey_01 | ITINERARY_PDF | 1 | 2026-01-10 12:40:00 | 2026-01-10 12:40:00 |

---

### 6. Supplementary Content & FAQs

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
        varchar   category    "INCLUDED | EXCLUDED | IMPORTANT_INFO"
        text      content
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    product_faqs {
        uuid      id          PK
        uuid      product_id  FK
        varchar   target_type "PRODUCT | VARIANT | TRIP"
        uuid      target_id   "polymorphic"
        text      question
        text      answer
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products ||--o{ product_supplementaries : "product_id"
    products ||--o{ product_faqs            : "product_id"
```

| Table | id | product_id | target_type | target_id | category | content | sort_order | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|
| `product_supplementaries` | supp_001 | prod_turkey_01 | PRODUCT | prod_turkey_01 | IMPORTANT_INFO | Valid passport with at least 6 months validity required from travel date. | 1 | 2026-01-10 13:00:00 | 2026-01-10 13:00:00 |
| `product_supplementaries` | supp_002 | prod_turkey_01 | PRODUCT | prod_turkey_01 | INCLUDED | 4-star hotel accommodations, domestic flights, daily breakfast, and English-speaking guide. | 2 | 2026-01-10 13:00:00 | 2026-01-10 13:00:00 |

| Table | id | product_id | target_type | target_id | question | answer | sort_order | created_at | updated_at |
|---|---|---|---|---|---|---|---|---|---|
| `product_faqs` | faq_001 | prod_turkey_01 | PRODUCT | prod_turkey_01 | Is the Cappadocia hot air balloon flight guaranteed? | Balloon flights are strictly subject to local civil aviation weather clearance. If cancelled, a full refund for the flight portion is provided. | 1 | 2026-01-10 13:15:00 | 2026-01-10 13:15:00 |

---

### 7. High-Level Domain Overview

```mermaid
flowchart LR
    subgraph TAXONOMY["📦 Taxonomy"]
        CAT["product_categories"]
    end

    subgraph CORE["🏷️ L1 — Core"]
        P["products"]
        PJ["product_journeys"]
    end

    subgraph HIERARCHY["🗂️ L2/L3 — Hierarchy"]
        PV["product_variants"]
        PT["product_trips"]
        PP["product_trip_pricings"]
    end

    subgraph CONTENT["📝 Content"]
        ITN["product_itineraries"]
        ITEM["product_itinerary_items"]
        LOC["product_locations"]
        SUPP["product_supplementaries"]
        FAQ["product_faqs"]
    end

    subgraph MEDIA["🖼️ Media"]
        M["product_media"]
        MU["product_media_usages"]
    end

    CAT -->|"1:N"| P
    P   -->|"1:1"| PJ
    P   -->|"1:N"| PV
    PV  -->|"1:N"| PT
    PT  -->|"1:N"| PP
    P   -->|"1:N"| ITN
    ITN -->|"1:N"| ITEM
    P   -->|"1:N"| LOC
    P   -->|"1:N"| SUPP
    P   -->|"1:N"| FAQ
    P   -->|"1:N"| M
    M   -->|"1:N"| MU

    MU  -."PRODUCT".-> P
    MU  -."VARIANT".-> PV
    MU  -."ITINERARY_ITEM".-> ITEM
    SUPP -."VARIANT".-> PV
    SUPP -."TRIP".-> PT
    FAQ  -."VARIANT".-> PV
    FAQ  -."TRIP".-> PT
```

---

## 📐 Index Summary

| Index Name | Table | Columns | Type | Purpose |
|---|---|---|---|---|
| `idx_products_status` | `products` | `(listing_status)` WHERE `deleted_at IS NULL` | B-Tree partial | Active product listing |
| `idx_products_slug_trgm` | `products` | `(slug)` | GIN pg_trgm | Destination text search |
| `idx_variants_product_id` | `product_variants` | `(product_id)` | B-Tree | Variant lookup by product |
| `idx_variants_name_trgm` | `product_variants` | `(name)` | GIN pg_trgm | Variant name text search |
| `idx_trips_variant_id` | `product_trips` | `(variant_id)` | B-Tree | Trip lookup by variant |
| `idx_trips_search` | `product_trips` | `(start_date, max_quota)` WHERE `status = 'ACTIVE'` | B-Tree partial | Search date+pax filter |
| `idx_locations_area_name_trgm` | `product_locations` | `(area_name)` | GIN pg_trgm | Destination text search |
| `idx_media_usages_target` | `product_media_usages` | `(target_type, target_id)` | B-Tree | Polymorphic media lookup |
| `uq_media_usages_product_itinerary_pdf` | `product_media_usages` | `(target_id, usage_context)` WHERE `target_type = 'PRODUCT' AND usage_context = 'ITINERARY_PDF'` | B-Tree unique partial | Enforce strict 1:1 single itinerary PDF per product |
| `idx_supplementaries_target` | `product_supplementaries` | `(target_type, target_id)` | B-Tree | Polymorphic content lookup |
| `idx_faqs_target` | `product_faqs` | `(target_type, target_id)` | B-Tree | Polymorphic FAQ lookup |
