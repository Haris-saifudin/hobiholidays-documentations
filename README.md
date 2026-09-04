# Hobiholidays — Technical Architecture & API Documentation

> **Central Engineering Documentation Repository**
> Comprehensive architectural blueprints, data models, PostgreSQL DDL migrations, and RESTful API contracts for the Hobiholidays tour package booking platform.
>
> _Engineered for High Scalability, Micro-Frontend modularity, NestJS + PostgreSQL stack, and Next.js App Router edge performance._

---

## 🗺️ Architectural Document Map

### 1. Core Technical Design Specifications

| Technical Design Document | Focus & Domain Responsibility |
| :--- | :--- |
| **[Product Technical Design](./product-technical-design.md)** | **Authoritative single source of truth for PostgreSQL DDL**: Products (L1), Journeys, Variants (L2), Trips (L3), Pricing tiers, Itineraries, Locations, Media, Supplementaries, and SEO tables with audit triggers and indexing summary. |
| **[Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)** | **3-level Product Hierarchy mental model** (`Product → Variant → Trip`), All Tours card rendering rules, relationship cardinality constraints, and real-world GWE catalog examples. |
| **[Area Domain Technical Design](./area-technical-design.md)** | **3-tier Geography tree** (`Continent → Country → City`, strictly capped at City level), adjacency list traversal, PostGIS spatial boundary models, and destination markers. |
| **[Search & Filter Architecture](./product-search-filter-technical-design.md)** | **Catalog Search Engine**: SQL execution strategy with multi-tier Area joins, DTO validation, and multi-criteria filters (**Continent, Country, Product Name, Total Pack / Pax, Min/Max Price, Departure Month, Variant Type**). |
| **[Product Media Technical Design](./product-media-technical-design.md)** | **2-Phase Progressive Storage Strategy**: Phase 1 (Database-First `BYTEA` storage & streaming) to Phase 2 (Cloud S3/R2 Object Store + Cloudflare CDN), polymorphic usage binding, and strict 1:1 Itinerary PDF brochure guarantee. |
| **[SEO Technical Design](./seo-technical-design.md)** | **Enterprise SEO Architecture**: Polymorphic `seo_metadata` table, dynamic programmatic fallbacks, Google Rich Results (**Schema.org `TouristTrip` + `Offer` + `BreadcrumbList`**), and Next.js App Router implementation (`generateMetadata`, `sitemap.ts`, `robots.ts`). |

---

### 2. API Contracts Suite (`contracts/`)

Complete REST API specifications defining endpoints, NestJS `class-validator` DTOs, request payloads, success/error envelopes, and TypeScript interfaces:

| Contract Document | Domain Coverage | Key Highlights |
| :--- | :--- | :--- |
| **[Contracts Master Guide](./contracts/README.md)** | Global Standards | Base URL `/api/v1`, RFC 7807 error responses, standard pagination envelopes, and HTTP status codes. |
| **[Product Contracts](./contracts/product-contract.md)** | Product Domain | **Split Sub-Resource GET Endpoints**: Dedicated granular retrieval for `/media`, `/itineraries`, `/locations`, `/variants`, `/supplementaries`, and `/seo` to eliminate over-fetching. |
| **[Product Hierarchy Contracts](./contracts/product-hierarchy-contract.md)** | Catalog & Details | All Tours listing feed (`GET /api/v1/variants`) and public aggregated Variant Detail page (`GET /api/v1/variants/:slug`) with embedded SEO. |
| **[Search & Filter Contracts](./contracts/product-search-filter-contract.md)** | Discovery Engine | `GET /api/v1/variants/search` query DTO, parameter validation rules, `totalPackages` metadata, and concrete real-world response payloads. |
| **[Product Media Contracts](./contracts/product-media-contract.md)** | Media Subsystem | Phase 1 multipart upload & binary streaming (`/stream`), Phase 2 presigned cloud uploads, polymorphic attachments, and itinerary brochure assignment. |
| **[Area Domain Contracts](./contracts/area-contract.md)** | Geography Domain | 3-tier hierarchy navigation (Continents, Countries, Cities), search widget autocomplete (`/autocomplete`), and destination landing pages. |

---

## 🏛️ Core Platform Architectural Highlights

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        3-Level Product Hierarchy                                │
│   Products (L1 Master Brand)                                                    │
│     └── Product Variants (L2 Bookable Card on All Tours)                        │
│           └── Product Trips (L3 Concrete Departure Window & Quota)              │
│                 └── Product Trip Pricings (Pricing by Nationality Scope)        │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┴────────────────────────────────────────┐
│                        3-Tier Area Hierarchy                                    │
│   Continents (Tier 1 Root)                                                      │
│     └── Countries (Tier 2 Sovereign States)                                     │
│           └── Cities (Tier 3 Maximum Granularity & Destination Marker)          │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┴────────────────────────────────────────┐
│                        2-Phase Progressive Media Storage                        │
│   Phase 1 (Database-First): product_media_blobs (BYTEA) + /stream endpoint      │
│   Phase 2 (Cloud Scale):    AWS S3 / Cloudflare R2 + Cloudflare Edge CDN        │
│   Zero Frontend Code Changes: Consumes media.url seamlessly across phases       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1. 3-Level Product Hierarchy
Tours are separated into three distinct structural layers:
- **Product (L1):** The master brand or program umbrella (e.g. *Grand West Europe*, *Turkey Wonders*). Owns binary media assets, itineraries, destination markers, and supplementary content.
- **Product Variant (L2):** The primary bookable tour card rendered on the All Tours listing page (e.g. *GWE Classic All-Year*, *GWE Spring 2026*, *Tulip Keukenhof Special*).
- **Product Trip (L3):** Specific departure date windows (e.g. *10 Apr 2026 – 20 Apr 2026*) tracking passenger capacity (`min_quota`, `max_quota`) and nationality-scoped pricing tiers (`ALL`, `DOMESTIC`, `INTERNATIONAL`).

### 2. 3-Tier Geography Domain
Geographic data follows a strict 3-tier closure pattern (`Continent → Country → City`), capping maximum granularity at the City level. Products anchor destination stops at the City level, while queries dynamically traverse up to Country and Continent levels for regional filtering and destination landing hubs.

### 3. Multi-Criteria Search & Filter Engine
The search engine executes a single unified query joining Variants, Products, Journey metadata, Area hierarchy, Trips, and Pricings. It supports:
- **Geographic Filtering:** By Continent (ID or slug) and Country (ID or slug).
- **Product Name Search:** Trigram GIN partial text search matching L1 Product and L2 Variant titles.
- **Total Pack / Pax Quota:** Passenger party size evaluation ensuring `min_quota <= totalPack AND max_quota >= totalPack`.
- **Budget Range:** `minPrice` and `maxPrice` filters evaluated against trip selling prices.
- **Result Counting:** In-query `COUNT(*) OVER() AS total_packages` window function for instant pagination and UI badges.

### 4. 2-Phase Progressive Media Storage
- **Phase 1 (Database-First):** Requires zero external cloud setup or cloud costs. Binary files are stored in PostgreSQL (`product_media_blobs`) and streamed with aggressive HTTP caching.
- **Phase 2 (Cloud Storage & CDN):** High-traffic production offloading to AWS S3 / Cloudflare R2 with Cloudflare CDN edge caching and WebP/AVIF dynamic transformation.
- **Zero Frontend Breaking Changes:** The frontend consumes `media.url` directly, maintaining identical behavior in both phases.

### 5. Enterprise SEO & Google Rich Results
- **Polymorphic Metadata (`seo_metadata`):** Supports custom overrides for Products, Variants, and Destinations with automatic programmatic fallback generation.
- **Schema.org Structured Data:** Google Rich Results support via `TouristTrip` (itinerary stops, departures), `Product` & `Offer` (pricing in IDR, availability), and `BreadcrumbList`.
- **Next.js App Router Integration:** Production-ready `generateMetadata()` implementation, dynamic XML sitemaps (`app/sitemap.ts`), and crawler directives (`app/robots.ts`).

---

## 💻 Technology Stack

- **Backend Framework:** NestJS (Node.js / TypeScript)
- **Database Engine:** PostgreSQL 16+ with extensions (`uuid-ossp`, `pg_trgm`, `postgis`)
- **ORM / Query Layer:** TypeORM / Kysely
- **Frontend Framework:** Next.js 15+ (App Router, Server Components, SSR/SSG/ISR)
- **Styling:** Vanilla CSS / Tailwind CSS
- **Object Storage & CDN:** AWS S3 / Cloudflare R2 + Cloudflare CDN
- **Validation:** NestJS `class-validator` and `class-transformer`
