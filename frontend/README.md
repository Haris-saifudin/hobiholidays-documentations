# Next.js 15 Frontend Architecture & Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Comprehensive frontend engineering guide for the Hobiholidays web application. Engineered for Next.js 15+ (App Router), React Server Components (RSC), Streaming SSR with React Suspense, `next/image` optimization across storage phases, dynamic SEO metadata, and fluid UI micro-interactions.
>
> _Target Audience: Frontend Engineers, Next.js Developers, UI/UX Specialists, and Web Performance Engineers._

---

## 🗺️ Frontend Guides Directory Map

| Guide Document | Domain Scope | Primary Frontend Implementation Focus |
| :--- | :--- | :--- |
| **[Product Frontend Guide](./product-frontend-guide.md)** | Product Detail Page (PDP) | PDP tabbed architecture, streaming sub-resource data fetching with React Suspense, official itinerary PDF brochure viewer, and breadcrumbs. |
| **[Product Hierarchy Frontend Guide](./product-hierarchy-frontend-guide.md)** | Catalog Feed & Cards | All Tours listing page (`/tours`), bookable Variant Card component, variant type badging system (`STANDARD`, `SEASONAL`, `THEMED`, `PROMOTIONAL`), and departure calendar picker. |
| **[Area Domain Frontend Guide](./area-frontend-guide.md)** | Discovery & Geography | "Where To?" search autocomplete widget (hero & navbar), 3-tier hierarchical dropdown (Continent/Country/City), and destination landing hubs (`/destinations/[slug]`). |
| **[Search & Filter Frontend Guide](./product-search-filter-frontend-guide.md)** | Catalog Search & Filter UI | Bidirectional URL query synchronization (`useSearchParams`, `useRouter`), filter sidebar (Price slider, Month picker, Total Pack counter), active chips, and skeleton states. |
| **[Product Media Frontend Guide](./product-media-frontend-guide.md)** | Media & Image Optimization | Complete `next/image` integration, `remotePatterns` configuration for Phase 1 stream API vs Phase 2 Cloudflare CDN, responsive image presets, and gallery carousel. |
| **[SEO Frontend Guide](./seo-frontend-guide.md)** | SEO & Google Rich Snippets | Dynamic `generateMetadata()` implementation, Schema.org JSON-LD structured data injection (`TouristTrip`, `Offer`, `BreadcrumbList`), dynamic `app/sitemap.ts`, and `app/robots.ts`. |

---

## 🏛️ Cross-Pillar References

- **Data Models & Schema:** [Technical Architecture](../technical/README.md) — Authoritative PostgreSQL DDL schemas, ERDs, triggers, and indexes.
- **API Interfaces:** [REST API Contracts](../contracts/README.md) — Endpoints, NestJS `class-validator` DTOs, and response envelopes.
- **Backend Mechanics:** [NestJS Backend Guides](../backend/README.md) — Service layer, transactions, and binary streaming.

---

## 🏗️ Next.js 15 App Router Architecture

### 1. Application Directory Layout

```
src/
├── app/
│   ├── layout.tsx                     # Root HTML layout & fonts
│   ├── page.tsx                       # Homepage with "Where To?" search widget
│   ├── sitemap.ts                     # Dynamic XML sitemap generator
│   ├── robots.ts                      # Dynamic crawler directives
│   ├── tours/
│   │   ├── page.tsx                   # All Tours catalog & filter view
│   │   └── [productSlug]/
│   │       └── [variantSlug]/
│   │           └── page.tsx           # Variant Detail Page (PDP)
│   └── destinations/
│       └── [slug]/
│           └── page.tsx               # Destination landing page hub
├── components/
│   ├── ui/                            # Atoms: Button, Badge, Modal, Input
│   ├── search/                        # SearchWidget, FilterSidebar, ActiveChips
│   ├── tour/                          # VariantCard, DeparturePicker, ItineraryTabs
│   └── media/                         # ResponsiveImage, GalleryCarousel, PdfViewer
├── hooks/
│   ├── use-debounce.ts
│   └── use-search-sync.ts
└── lib/
    ├── api-client.ts                  # Typed fetch client with error handling
    └── formatters.ts                  # Currency IDR, date formatting
```

### 2. Server vs Client Component Boundary
To guarantee minimal JavaScript bundle sizes and ultra-fast Core Web Vitals (LCP < 1.2s):
- **Server Components (Default):** All page layouts, initial catalog fetches, PDP static content, and `generateMetadata()` execute on the server.
- **Client Components (`'use client'`):** Restricted strictly to interactive leaves: autocomplete search dropdowns, date pickers, filter sliders, image carousels, and booking modal wizards.

### 3. Data Fetching & Caching Strategy
The frontend utilizes Next.js native `fetch()` extensions with Incremental Static Regeneration (ISR):

```typescript
// lib/api-client.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'https://api.hobiholidays.com/api/v1';

export async function fetchApi<T>(endpoint: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${API_BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!res.ok) {
    const errorBody = await res.json().catch(() => ({}));
    throw new Error(errorBody.message || `API request failed with status ${res.status}`);
  }

  const json = await res.json();
  return json.data;
}
```
