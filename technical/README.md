# Technical Architecture & System Design Specifications

> **Pillar 1: Pure Technical Architecture**
> Comprehensive architectural blueprints, relational data models, authoritative PostgreSQL 16+ DDL schemas, ERDs, triggers, constraints, indexing strategies, and concurrency control models for the Hobiholidays tour package booking platform.
>
> _Target Audience: System Architects, Database Administrators (DBAs), Solutions Architects, and DevOps Engineers._

---

## 🗺️ Technical Specifications Map

| Document | Domain Scope | Primary Architectural Focus |
| :--- | :--- | :--- |
| **[Product Technical Design](./product-technical-design.md)** | Core Catalog & Entities | **Authoritative single source of truth for PostgreSQL DDL**: Multi-Dimensional Taxonomy (`category_dimensions`, `product_categories`, `product_category_assignments`), Products (L1), Journeys, Variants (L2), Trips (L3), Pricing tiers (`product_trip_pricings`), Itemized bundled breakdown components (`product_pricing_components`), Add-ons (`product_addons`), Itineraries, Locations, Media, Supplementaries, and SEO tables. Includes audit triggers and composite indexes. |
| **[Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)** | Catalog Structural Model | **3-level Product Hierarchy mental model** (`Product → Variant → Trip → Pricing Tier → Pricing Components`), Multi-Dimensional taxonomy tagging & variant overrides, catalog lifecycle management (`listing_status`), duration inheritance resolution, nominal capacity indicators, and real-world GWE catalog examples. |
| **[Area Domain Technical Design](./area-technical-design.md)** | Geographic Hierarchy | **4-tier Geography tree** (`Continent → Sub Continent → Country → POI`), adjacency list pattern, pure relational hierarchy, flexible flat anchoring (POI, Country, Sub-Continent, Continent), and dynamic upward hierarchy traversal. |
| **[Search & Filter Architecture](./product-search-filter-technical-design.md)** | Discovery Engine | **Catalog Search Mechanics**: High-performance relational join strategy with dynamic upward Area traversal, PostgreSQL `pg_trgm` GIN indexes, window function result counting (`COUNT(*) OVER()`), and execution plan optimization. |
| **[Product Media Technical Design](./product-media-technical-design.md)** | Media Asset Subsystem | **2-Phase Progressive Storage Architecture**: Phase 1 (Database-First `BYTEA` storage & streaming) to Phase 2 (Cloud S3/R2 Object Store + Cloudflare CDN), polymorphic visual asset binding (`IMAGE` and `VIDEO`). Official tour itinerary PDF brochures are compiled externally by ATW and referenced directly via `itinerary_pdf_url`. |
| **[SEO Technical Design](./seo-technical-design.md)** | Discovery & Rich Snippets | **Polymorphic Metadata Architecture**: `seo_metadata` schema, dynamic programmatic fallback formulas, and Schema.org graph architectures (`TouristTrip`, `Product`, `Offer`, `BreadcrumbList`). |

---

## 🏛️ Cross-Pillar References

To explore how these technical specifications are consumed across the platform, consult the companion pillars:
- **API Interfaces:** [REST API Contracts](../contracts/README.md) — Endpoint routes, NestJS `class-validator` DTOs, and standard response envelopes.
- **Backend Implementation:** [NestJS Backend Guides](../backend/README.md) — Modules, services, controllers, Kysely/TypeORM query builders, transactions, and migration scripts.
- **Frontend Implementation:** [Next.js Frontend Guides](../frontend/README.md) — Next.js 15 App Router architecture, `next/image` setup, dynamic metadata, and UI components.

---

## ⚙️ Core Database Standards & Principles

All data models within this repository adhere to the following PostgreSQL 16+ engineering standards:

### 1. Primary Keys & Identifiers
- All tables use UUID v4 primary keys generated via `DEFAULT uuid_generate_v4()`.
- Relational integrity is strictly enforced with foreign key constraints configured for non-destructive operations (`ON DELETE RESTRICT` / `ON DELETE SET NULL`).

### 2. Temporal & Audit Tracking & Safe Idempotent Lifecycle
- All stateful tables contain `created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` and `updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP`.
- Row mutations automatically invoke the database trigger function `set_updated_at_timestamp()` before update.
- Core catalog entities (`products`, `product_variants`, `areas`) enforce soft deletion via nullable `deleted_at TIMESTAMP NULL` and state machine governance via `listing_status` (`'ACTIVE'`, `'INACTIVE'`, `'ARCHIVED'`). Catalog synchronization from ATW is non-destructive and idempotent, eliminating the need for destructive cascading database drops (`DELETE CASCADE`).

### 3. Decoupled PostGIS & Pure Relational Geography
- All PostGIS extensions, geometry columns (`GEOMETRY`), GiST spatial indexes, and spatial query operators (`ST_Contains`, `ST_Within`) are strictly eliminated.
- Hierarchy classification and navigation use pure relational B-Tree indexing on `(parent_id, area_type_id, slug)` and trigram text search.
- The platform retains only standard PostgreSQL extensions: `"uuid-ossp"` and `"pg_trgm"`.

### 4. Decoupled Booking Concurrency & Read-Only Nominal Availability
- Catalog discovery endpoints and PDP departure calendars surface nominal seat availability metrics:
  $$\text{availableSeats} = \max(0, \text{max\_quota} - \text{booked\_seats})$$
- Transactional pessimistic locks (`SELECT ... FOR UPDATE`), mutex quota deductions, and lock TTL mechanisms are decoupled from the catalog domain and delegated downstream to Phase 3 (Booking & Checkout Domain).

### 5. Enumerations & Domain Constraints
- Enumerations are enforced at the database level using `CHECK (column_name IN ('VALUE1', 'VALUE2'))` constraints rather than native PostgreSQL ENUM types, enabling seamless zero-downtime alterations without type recreation.

### 6. Indexing & Query Acceleration
- Geographic hierarchy traversal utilizes composite B-Tree indexes (`(parent_id, area_type_id, slug)`).
- Text search utilizes PostgreSQL Trigram GIN indexes (`gin (name gin_trgm_ops)`).
- Polymorphic target references use composite B-Tree indexes (`(target_type, target_id)`).
- Partial indexes filter active records (`WHERE deleted_at IS NULL` or `WHERE status = 'ACTIVE'`).
