# Hobiholidays — Technical Architecture & Engineering Documentation

> **Central Engineering Documentation Repository**
> Architectural blueprints, relational PostgreSQL 16+ data models, RESTful API contracts, NestJS backend implementation guides, and Next.js 15 App Router frontend specifications for the Hobiholidays tour package booking platform.
>
> _Engineered for high scalability, micro-frontend modularity, NestJS + PostgreSQL stack, and Next.js App Router edge performance._

---

## 🏛️ Platform Documentation Architecture (4 Pillars)

The documentation suite is structured into four role-specialized architectural pillars:

```
hobiholidays-documentations/
├── technical/                             # PILLAR 1: Pure Technical Architecture & Data Models
│   ├── README.md                          # Technical standards, PostgreSQL DDL conventions, ERDs
│   ├── product-technical-design.md        # Authoritative PostgreSQL DDL, core ERD, audit triggers
│   ├── product-hierarchy-technical-design.md # 3-Level hierarchy mental model, cascade rules, quota
│   ├── area-technical-design.md           # 4-Tier geography tree (Continent → Sub Continent → Country → POI), PostGIS
│   ├── product-search-filter-technical-design.md # SQL join mechanics, trigram GIN, window functions
│   ├── product-media-technical-design.md  # 2-Phase storage architecture, binary blobs, 1:1 PDF
│   └── seo-technical-design.md            # Polymorphic SEO schema, formula matrix, rich snippets
│
├── contracts/                             # PILLAR 2: REST API Contracts & DTOs
│   ├── README.md                          # Global standards, base URL, pagination, RFC 7807 errors
│   ├── product-contract.md                # Base Product CRUD, split sub-resources, trips, pricings
│   ├── product-hierarchy-contract.md      # All Tours catalog feed, variant detail view contract
│   ├── area-contract.md                   # Area types, Continents/Sub Continents/Countries/POIs, autocomplete
│   ├── product-search-filter-contract.md  # Search DTO, filters, paginated response, totalPackages
│   ├── product-media-contract.md          # Multipart upload, streaming, S3 presigned, usages
│   └── seo-contract.md                    # Polymorphic SEO metadata CRUD & bulk retrieval
│
├── backend/                               # PILLAR 3: NestJS Backend Implementation Guides
│   ├── README.md                          # NestJS architecture, module graph, filters, interceptors
│   ├── product-backend-guide.md           # ProductModule, split sub-resource service architecture
│   ├── product-hierarchy-backend-guide.md # Duration inheritance COALESCE, pessimistic booking lock
│   ├── area-backend-guide.md              # Recursive CTE traversal query, PostGIS lookup, cache
│   ├── product-search-filter-backend-guide.md # Dynamic SQL builder, trigram search, offset pagination
│   ├── product-media-backend-guide.md     # Multer BYTEA streaming controller, S3 migration script
│   └── seo-backend-guide.md               # Dynamic fallback resolver service, JSON-LD generator
│
└── frontend/                              # PILLAR 4: Next.js Frontend Implementation Guides
    ├── README.md                          # Next.js 15 App Router structure, RSC vs Client boundary
    ├── product-frontend-guide.md          # PDP tabs, split sub-resource fetching, brochure download
    ├── product-hierarchy-frontend-guide.md# All Tours catalog card, category & variant badging, Adult price
    ├── area-frontend-guide.md             # Where To? autocomplete widget, destination landing pages
    ├── product-search-filter-frontend-guide.md # URL query sync (useSearchParams), filter sidebar
    ├── product-media-frontend-guide.md    # next/image remotePatterns, responsive presets, blurhash
    └── seo-frontend-guide.md              # generateMetadata(), JSON-LD injection, sitemap, robots
```

---

## 🗺️ Master Cross-Reference Matrix

Every core domain in the Hobiholidays platform maps across all four pillars:

| Domain Area | Technical Architecture (`technical/`) | REST API Contract (`contracts/`) | NestJS Backend Guide (`backend/`) | Next.js Frontend Guide (`frontend/`) |
| :--- | :--- | :--- | :--- | :--- |
| **Global Standards** | [Technical README](./technical/README.md) | [Contracts README](./contracts/README.md) | [Backend README](./backend/README.md) | [Frontend README](./frontend/README.md) |
| **Product Core & Catalog** | [product-technical-design.md](./technical/product-technical-design.md) | [product-contract.md](./contracts/product-contract.md) | [product-backend-guide.md](./backend/product-backend-guide.md) | [product-frontend-guide.md](./frontend/product-frontend-guide.md) |
| **Product Hierarchy** | [product-hierarchy-technical-design.md](./technical/product-hierarchy-technical-design.md) | [product-hierarchy-contract.md](./contracts/product-hierarchy-contract.md) | [product-hierarchy-backend-guide.md](./backend/product-hierarchy-backend-guide.md) | [product-hierarchy-frontend-guide.md](./frontend/product-hierarchy-frontend-guide.md) |
| **Area & Geography** | [area-technical-design.md](./technical/area-technical-design.md) | [area-contract.md](./contracts/area-contract.md) | [area-backend-guide.md](./backend/area-backend-guide.md) | [area-frontend-guide.md](./frontend/area-frontend-guide.md) |
| **Search & Discovery** | [product-search-filter-technical-design.md](./technical/product-search-filter-technical-design.md) | [product-search-filter-contract.md](./contracts/product-search-filter-contract.md) | [product-search-filter-backend-guide.md](./backend/product-search-filter-backend-guide.md) | [product-search-filter-frontend-guide.md](./frontend/product-search-filter-frontend-guide.md) |
| **Media Subsystem** | [product-media-technical-design.md](./technical/product-media-technical-design.md) | [product-media-contract.md](./contracts/product-media-contract.md) | [product-media-backend-guide.md](./backend/product-media-backend-guide.md) | [product-media-frontend-guide.md](./frontend/product-media-frontend-guide.md) |
| **SEO & Rich Snippets** | [seo-technical-design.md](./technical/seo-technical-design.md) | [seo-contract.md](./contracts/seo-contract.md) | [seo-backend-guide.md](./backend/seo-backend-guide.md) | [seo-frontend-guide.md](./frontend/seo-frontend-guide.md) |

---

## 🚀 Platform Roadmap & Architecture Phasing

The platform engineering lifecycle is organized into structured development phases:

### Phase 1: Core Catalog, Discovery & SEO (Current Scope — Authoritative & Complete)
The documentation in this repository currently defines the complete, authoritative specification for **Phase 1**:
- **Product Core & Master Catalog:** Master brand umbrella (`products`), 2-tier Category taxonomy, journey durations, destination markers, and supplementary inclusions.
- **Product Hierarchy:** 3-Level hierarchy (`Product → Variant → Trip → Pricing`), duration inheritance (`COALESCE`), pessimistic locking (`SELECT FOR UPDATE`), relational promotional badges (`product_badges`), and itemized cost breakdown components (`product_pricing_components`).
- **Area & Geography:** 4-Tier geographic taxonomy (`Continent → Sub Continent → Country → POI`), PostGIS spatial coordinates, recursive CTE tree traversal, and "Where To?" autocomplete.
- **Search & Dynamic Discovery:** Trigram fuzzy matching (`pg_trgm`), windowed total counts (`COUNT(*) OVER()`), and dynamic active-only filter options aggregation (`/api/v1/variants/search/filter-options`).
- **Media Subsystem:** 2-Phase progressive storage (Database BYTEA binary blobs for zero-dependency local dev vs Cloud AWS S3/Cloudflare R2 presigned URLs).
- **SEO & Rich Snippets:** Polymorphic SEO schema (`seo_metadata`), programmatic dynamic fallbacks, Google Rich Results (Schema.org `TouristTrip` & `Offer`), dynamic sitemaps, and robots.txt.

---

### Subsequent Development Phases (Scheduled Roadmap)
The following e-commerce domains are intentionally scheduled for subsequent phases:
- **Phase 2: Authentication, Authorization & User Management**
  - AWS Cognito User Pool integration, custom password API endpoints, better-auth session cookies, and RBAC guards (`CUSTOMER`, `AGENT`, `ADMIN`).
- **Phase 3: Booking & Reservation Engine**
  - Booking drafts (`POST /api/v1/bookings/draft`), dynamic checkout forms, lead passenger & companion validation, dynamic questions, and pessimistic quota reservation TTL.
- **Phase 4: Payments & Transaction Processing**
  - Payment gateway adapters (Duitku, Midtrans, Xendit), QRIS dynamic image generation & countdown, Virtual Accounts, webhook HMAC verification, and payment status synchronization.
- **Phase 5: Customer Portal, Ancillaries & Community**
  - Traveler portal ("My Bookings"), PDF travel vouchers & invoices, airport lounge reservations, and verified traveler reviews & ratings (`TouristTrip` UGC).

---

## 🎯 Role-Based Reading Paths

Choose your entry point based on your engineering discipline:

### 1. For Solutions Architects & Database Administrators (DBAs)
1. Start with **[Technical Architecture README](./technical/README.md)** for global DDL conventions, trigger strategies, and data modeling standards.
2. Review the **[Product Technical Design](./technical/product-technical-design.md)** for the authoritative PostgreSQL DDL schemas.
3. Understand the **[Product Hierarchy Technical Design](./technical/product-hierarchy-technical-design.md)** for cascade integrity and quota concurrency models.

### 2. For Backend & API Platform Engineers
1. Read the **[Backend README](./backend/README.md)** for NestJS modular structure, validation pipes, and RFC 7807 error handling.
2. Review the **[Contracts README](./contracts/README.md)** for global endpoint standards and pagination envelopes.
3. Dive into the domain-specific backend guides:
   - **[Product Backend Guide](./backend/product-backend-guide.md)** — Split sub-resource architecture.
   - **[Hierarchy Backend Guide](./backend/product-hierarchy-backend-guide.md)** — Duration `COALESCE` and pessimistic booking lock (`SELECT FOR UPDATE`).
   - **[Media Backend Guide](./backend/product-media-backend-guide.md)** — Database streaming and AWS S3 presigned uploads.

### 3. For Frontend & UI/UX Engineers
1. Read the **[Frontend README](./frontend/README.md)** for Next.js 15 App Router architecture, Server vs Client component boundaries, and ISR caching.
2. Explore UI implementation guides:
   - **[Product Frontend Guide](./frontend/product-frontend-guide.md)** — Variant detail tabs, React Suspense streaming, and PDF brochure downloader.
   - **[Product Hierarchy Frontend Guide](./frontend/product-hierarchy-frontend-guide.md)** — Bookable Variant Card and visual badging tokens.
   - **[Search & Filter Frontend Guide](./frontend/product-search-filter-frontend-guide.md)** — URL query sync (`useSearchParams`) and filter sidebar.
   - **[Media Frontend Guide](./frontend/product-media-frontend-guide.md)** — `next/image` setup for Phase 1 vs Phase 2 and responsive presets.
   - **[SEO Frontend Guide](./frontend/seo-frontend-guide.md)** — `generateMetadata()` and Schema.org JSON-LD structured data.

### 4. For QA Engineers & Third-Party Integrators
1. Consult **[Contracts README](./contracts/README.md)** for base URLs, authentication headers, and standard response envelopes.
2. Refer to individual contracts in **[`contracts/`](./contracts/)** for complete DTO validation rules, query parameters, and concrete JSON payloads.

---

## 💻 Technology Stack Summary

- **Backend Framework:** NestJS (Node.js / TypeScript)
- **Database Engine:** PostgreSQL 16+ with extensions (`uuid-ossp`, `pg_trgm`, `postgis`)
- **Query & ORM Layer:** TypeORM / Kysely
- **Frontend Framework:** Next.js 15+ (App Router, Server Components, SSR/SSG/ISR)
- **Image Optimization:** Next.js Image Component (`next/image`) with WebP/AVIF auto-negotiation
- **Object Storage & CDN:** AWS S3 / Cloudflare R2 + Cloudflare Edge CDN
- **Validation:** NestJS `class-validator` and `class-transformer`
