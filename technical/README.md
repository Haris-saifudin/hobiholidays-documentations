# Technical Architecture & System Design Specifications

> **Pillar 1: Pure Technical Architecture**
> Comprehensive architectural blueprints, relational data models, authoritative PostgreSQL 16+ DDL schemas, ERDs, triggers, constraints, indexing strategies, and concurrency control models for the Hobiholidays tour package booking platform.
>
> _Target Audience: System Architects, Database Administrators (DBAs), Solutions Architects, and DevOps Engineers._

---

## 🗺️ Technical Specifications Map

| Document | Domain Scope | Primary Architectural Focus |
| :--- | :--- | :--- |
| **[Product Technical Design](./product-technical-design.md)** | Core Catalog & Entities | **Authoritative single source of truth for PostgreSQL DDL**: Products (L1), Journeys, Variants (L2), Trips (L3), Pricing tiers (L3+), Itineraries, Locations, Media, Supplementaries, and SEO tables. Includes audit triggers and composite indexes. |
| **[Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)** | Catalog Structural Model | **3-level Product Hierarchy mental model** (`Product → Variant → Trip → Pricing`), cascade integrity, duration inheritance resolution, quota concurrency models (`SELECT FOR UPDATE`), and real-world GWE catalog examples. |
| **[Area Domain Technical Design](./area-technical-design.md)** | Geographic Hierarchy | **4-tier Geography tree** (`Continent → Sub Continent → Country → POI`), adjacency list pattern, PostGIS spatial models, and destination marker relations. |
| **[Search & Filter Architecture](./product-search-filter-technical-design.md)** | Discovery Engine | **Catalog Search Mechanics**: High-performance relational join strategy, PostgreSQL `pg_trgm` GIN indexes, window function result counting (`COUNT(*) OVER()`), and execution plan optimization. |
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
- Explicit foreign key constraints are strictly enforced with `ON DELETE CASCADE` or `ON DELETE RESTRICT` based on entity lifecycle ownership.

### 2. Temporal & Audit Tracking
- All stateful tables contain `created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP` and `updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP`.
- Row mutations automatically invoke the database trigger function `set_updated_at_timestamp()` before update.

### 3. Enumerations & Domain Constraints
- Enumerations are enforced at the database level using `CHECK (column_name IN ('VALUE1', 'VALUE2'))` constraints rather than native PostgreSQL ENUM types, enabling seamless zero-downtime alterations without type recreation.

### 4. Indexing & Query Acceleration
- Foreign key columns are explicitly indexed to guarantee fast cascade deletions and JOIN operations.
- Text search utilizes PostgreSQL Trigram GIN indexes (`gin (name gin_trgm_ops)`).
- Polymorphic target references use composite B-Tree indexes (`(target_type, target_id)`).
