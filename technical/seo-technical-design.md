# SEO Technical Design & Implementation Architecture

> **Overview**
> Technical documentation for Search Engine Optimization (SEO), Social Sharing (Open Graph / Twitter Cards), and Google Rich Results (Schema.org Structured Data) across the Hobiholidays tour platform.
>
> _Engineered for Next.js App Router (SSR/SSG), High-Converting Google Search Snippets (TouristTrip & Offer schema), Dynamic Programmatic Fallbacks, and Zero Cumulative Layout Shift._

> **See Also:**
> - [Product Technical Design](./product-technical-design.md)
> - [Product Hierarchy Technical Design](./product-hierarchy-technical-design.md)
> - [Product Media Technical Design](./product-media-technical-design.md) — OG image asset strategy
> - [Product Contracts](../contracts/product-contract.md) — SEO sub-resource endpoint
> - [Area Contracts](../contracts/area-contract.md) — Destination landing page with embedded SEO
> - [Backend Guide](../backend/seo-backend-guide.md) — Dynamic fallback resolver and Schema.org JSON-LD generator
> - [Frontend Guide](../frontend/seo-frontend-guide.md) — Next.js generateMetadata(), sitemap.ts, and robots.ts

---

## 🏗️ Architectural Principles & SEO Strategy

In travel and e-commerce platforms, organic search is the primary driver of high-intent bookings. The SEO architecture rests on five fundamental pillars:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        1. Semantic Canonical Routing                            │
│   • Clean, human-readable URLs: /tours/[product-slug]/[variant-slug]             │
│   • Strict 301 redirects on slug changes                                        │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┴────────────────────────────────────────┐
│               2. Dual-Layer Metadata (Custom DB + Fallback)                     │
│   • Layer A: Explicit admin overrides stored in PostgreSQL seo_metadata         │
│   • Layer B: Formulaic dynamic fallback if custom SEO is blank                  │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┴────────────────────────────────────────┐
│               3. Google Rich Results (Schema.org JSON-LD)                       │
│   • TouristTrip: Itinerary stops, departure dates, duration                      │
│   • Product & Offer: Price in IDR, availability (InStock / OutOfStock)          │
│   • BreadcrumbList: Crawlable hierarchical site trails                          │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
┌────────────────────────────────────────┴────────────────────────────────────────┐
│               4. Next.js App Router Server-Side Execution                       │
│   • generateMetadata() injected in <head> before page render                    │
│   • Dynamic XML Sitemaps (app/sitemap.ts) crawling active tours & areas         │
│   • Strict indexation guard (drafts/archived items emit noindex, nofollow)      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ PostgreSQL DDL: `seo_metadata` Table

To avoid polluting business entity tables while providing flexible metadata overrides for Products, Variants, and Destination Areas, we use a dedicated polymorphic table:

```sql
-- =========================================================================
-- SEO METADATA — Polymorphic metadata overrides
-- Binds to PRODUCT | VARIANT | AREA
-- =========================================================================
CREATE TABLE seo_metadata (
    id               UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
    target_type      VARCHAR(50)  NOT NULL,                  -- PRODUCT | VARIANT | AREA
    target_id        UUID         NOT NULL,                  -- Polymorphic FK
    meta_title       VARCHAR(255),                          -- Custom <title> tag (max 60 chars recommended)
    meta_description TEXT,                                  -- Custom <meta name="description"> (max 160 chars)
    canonical_url    VARCHAR(500),                          -- Explicit canonical override (defaults to current route)
    og_title         VARCHAR(255),                          -- Open Graph title (defaults to meta_title)
    og_description   TEXT,                                  -- Open Graph description (defaults to meta_description)
    og_image_url     VARCHAR(500),                          -- 1200x630 social share banner
    no_index         BOOLEAN      NOT NULL DEFAULT FALSE,   -- If TRUE, injects <meta name="robots" content="noindex">
    no_follow        BOOLEAN      NOT NULL DEFAULT FALSE,   -- If TRUE, injects <meta name="robots" content="nofollow">
    structured_data  JSONB,                                 -- Custom Schema.org JSON-LD overrides (optional)
    created_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_seo_target_type CHECK (target_type IN ('PRODUCT', 'VARIANT', 'AREA')),
    CONSTRAINT uq_seo_target       UNIQUE (target_type, target_id)
);

CREATE INDEX idx_seo_target ON seo_metadata(target_type, target_id);

CREATE TRIGGER trg_seo_metadata_updated_at
    BEFORE UPDATE ON seo_metadata
    FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
```

> [!NOTE]
> The `set_updated_at_timestamp()` function and this trigger are also defined in the **authoritative DDL** in [Product Technical Design](./product-technical-design.md#-postgresql-ddl-schema). The DDL above is reproduced here for standalone readability.

---

## ⚡ Programmatic Metadata Generation Engine

Content editors often do not write custom SEO tags for every single package. When `seo_metadata` fields are `NULL`, the backend / frontend generates keyword-rich, localized metadata using standardized fallback formulas:

### Formula Matrix

| Page Type | Route Pattern | Dynamic Title Formula | Dynamic Description Formula |
| :--- | :--- | :--- | :--- |
| **All Tours** | `/tours` | `Paket Tour & Liburan Keliling Dunia Terbaik \| Hobiholidays` | `Temukan paket tour wisata internasional & domestik terlengkap dengan jadwal pasti, fasilitas premium, dan harga transparan di Hobiholidays.` |
| **Tour Variant (Card/Detail)** | `/tours/[product]/[variant]` | `{variant_name} ({duration_days}D/{duration_nights}N) - Tour {destinations} \| Hobiholidays` | `Ikuti paket tour {variant_name}. Nikmati perjalanan {duration_days} hari mengunjungi {destinations}. Berangkat {next_departure}. Mulai IDR {starting_price}.` |
| **Continent Hub** | `/destinations/[continent]` | `Paket Tour {continent_name} Terlengkap & Promo Terbaru \| Hobiholidays` | `Jelajahi keindahan benua {continent_name}. Pilihan tour liburan terbaik ke berbagai negara dengan pemandu berpengalaman.` |
| **Country Hub** | `/destinations/[continent]/[country]`| `Paket Tour Wisata {country_name} - Liburan Impian \| Hobiholidays` | `Daftar paket tour ke {country_name} terlengkap. Jelajahi kota favorit, jadwal keberangkatan terupdate, dan booking mudah di Hobiholidays.` |

---

## 🌟 Google Rich Results (Schema.org JSON-LD)

Structured data is directly injected into the HTML as `<script type="application/ld+json">` during Server-Side Rendering (SSR). This enables Google to render rich snippets with pricing badges, departure schedules, and star ratings.

### 1. Tour Package Schema (`TouristTrip` & `Product` / `Offer`)

```json
{
  "@context": "https://schema.org",
  "@graph": [
    {
      "@type": "TouristTrip",
      "@id": "https://www.hobiholidays.com/tours/grand-west-europe/gwe-spring-2026#trip",
      "name": "GWE Spring 2026",
      "description": "Paket tour 11 hari 9 malam menjelajahi keindahan Eropa Barat di musim semi.",
      "touristType": "Leisure travelers",
      "offers": {
        "@type": "Offer",
        "price": "28000000",
        "priceCurrency": "IDR",
        "priceValidUntil": "2026-12-31",
        "availability": "https://schema.org/InStock",
        "url": "https://www.hobiholidays.com/tours/grand-west-europe/gwe-spring-2026",
        "validFrom": "2026-04-10"
      },
      "itinerary": {
        "@type": "ItemList",
        "numberOfItems": 2,
        "itemListElement": [
          {
            "@type": "ListItem",
            "position": 1,
            "item": {
              "@type": "TouristAttraction",
              "name": "Amsterdam Canal Cruise & Zaanse Schans",
              "address": {
                "@type": "PostalAddress",
                "addressLocality": "Amsterdam",
                "addressCountry": "NL"
              }
            }
          },
          {
            "@type": "ListItem",
            "position": 2,
            "item": {
              "@type": "TouristAttraction",
              "name": "Eiffel Tower & Louvre Museum",
              "address": {
                "@type": "PostalAddress",
                "addressLocality": "Paris",
                "addressCountry": "FR"
              }
            }
          }
        ]
      }
    },
    {
      "@type": "BreadcrumbList",
      "@id": "https://www.hobiholidays.com/tours/grand-west-europe/gwe-spring-2026#breadcrumb",
      "itemListElement": [
        {
          "@type": "ListItem",
          "position": 1,
          "name": "Beranda",
          "item": "https://www.hobiholidays.com"
        },
        {
          "@type": "ListItem",
          "position": 2,
          "name": "All Tours",
          "item": "https://www.hobiholidays.com/tours"
        },
        {
          "@type": "ListItem",
          "position": 3,
          "name": "Grand West Europe",
          "item": "https://www.hobiholidays.com/tours/grand-west-europe"
        },
        {
          "@type": "ListItem",
          "position": 4,
          "name": "GWE Spring 2026",
          "item": "https://www.hobiholidays.com/tours/grand-west-europe/gwe-spring-2026"
        }
      ]
    }
  ]
}
```

---

## 💻 Next.js App Router Implementation Guide

### 1. Dynamic Metadata (`app/tours/[productSlug]/[variantSlug]/page.tsx`)

```tsx
import type { Metadata } from 'next';
import { notFound } from 'next/navigation';

interface Props {
  params: { productSlug: string; variantSlug: string };
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const res = await fetch(
    `https://api.hobiholidays.com/api/v1/variants/${params.variantSlug}`,
    { next: { revalidate: 3600 } }, // ISR: Revalidate hourly
  );

  if (!res.ok) return {};
  const { data: variant } = await res.json();

  const title =
    variant.seo?.metaTitle ||
    `${variant.name} (${variant.durationDays}D/${variant.durationNights}N) | Hobiholidays`;

  const description =
    variant.seo?.metaDescription ||
    `Ikuti tour ${variant.name} mengunjungi ${variant.destinations.map((d: any) => d.city).join(', ')}. Mulai IDR ${new Intl.NumberFormat('id-ID').format(variant.startingPrice)}.`;

  const canonicalUrl =
    variant.seo?.canonicalUrl ||
    `https://www.hobiholidays.com/tours/${params.productSlug}/${params.variantSlug}`;

  const ogImage =
    variant.seo?.ogImageUrl ||
    variant.coverUrl ||
    'https://cdn.hobiholidays.com/defaults/og-tour-default.jpg';

  // Strict indexation guard: archived or inactive items are excluded from Google
  const isIndexable =
    variant.listingStatus === 'ACTIVE' && !variant.seo?.noIndex;

  return {
    title,
    description,
    alternates: {
      canonical: canonicalUrl,
    },
    robots: {
      index: isIndexable,
      follow: isIndexable,
      googleBot: {
        index: isIndexable,
        follow: isIndexable,
        'max-image-preview': 'large',
        'max-snippet': -1,
      },
    },
    openGraph: {
      title,
      description,
      url: canonicalUrl,
      siteName: 'Hobiholidays',
      images: [
        {
          url: ogImage,
          width: 1200,
          height: 630,
          alt: title,
        },
      ],
      type: 'website',
      locale: 'id_ID',
    },
    twitter: {
      card: 'summary_large_image',
      title,
      description,
      images: [ogImage],
    },
  };
}
```

---

### 2. Dynamic XML Sitemap (`app/sitemap.ts`)

Next.js automatically serves an optimized, segmented XML sitemap at `/sitemap.xml`:

```typescript
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const BASE_URL = 'https://www.hobiholidays.com';

  // 1. Fetch all active public variants
  const variantsRes = await fetch('https://api.hobiholidays.com/api/v1/variants?limit=500');
  const { data: variants } = await variantsRes.json();

  // 2. Fetch all active destination areas (Continents & Countries)
  const areasRes = await fetch('https://api.hobiholidays.com/api/v1/areas/sitemap');
  const { data: areas } = await areasRes.json();

  const tourUrls = variants.map((v: any) => ({
    url: `${BASE_URL}/tours/${v.productSlug}/${v.slug}`,
    lastModified: new Date(v.updatedAt),
    changeFrequency: 'daily' as const,
    priority: 0.9,
  }));

  const areaUrls = areas.map((a: any) => ({
    url: `${BASE_URL}/destinations/${a.slug}`,
    lastModified: new Date(a.updatedAt),
    changeFrequency: 'weekly' as const,
    priority: 0.8,
  }));

  return [
    {
      url: BASE_URL,
      lastModified: new Date(),
      changeFrequency: 'daily',
      priority: 1.0,
    },
    {
      url: `${BASE_URL}/tours`,
      lastModified: new Date(),
      changeFrequency: 'daily',
      priority: 0.95,
    },
    ...tourUrls,
    ...areaUrls,
  ];
}
```

---

### 3. Dynamic Crawler Directives (`app/robots.ts`)

```typescript
// app/robots.ts
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: [
          '/api/',
          '/admin/',
          '/checkout/',
          '/booking/confirmation',
          '/*?*pax=*', // Avoid duplicate crawler indexing on query parameter combinations
        ],
      },
    ],
    sitemap: 'https://www.hobiholidays.com/sitemap.xml',
  };
}
```

---

## 🔄 Canonical URL & 301 Redirect Architecture

When a product or variant title is modified by an administrator, the slug may change (e.g. `gwe-spring` → `gwe-spring-2026`). To prevent Google 404 penalties and preserve PageRank:

1. **Slugs Table / Historical Slugs:** When updating a slug, the old slug is recorded in `slug_redirects (old_path, new_path, status_code: 301)`.
2. **Next.js Middleware Redirect:**
   ```typescript
   // middleware.ts
   import { NextResponse } from 'next/server';
   import type { NextRequest } from 'next/server';

   export async function middleware(req: NextRequest) {
     const pathname = req.nextUrl.pathname;
     const redirect = await checkHistoricalRedirect(pathname);
     if (redirect) {
       return NextResponse.redirect(new URL(redirect.newPath, req.url), 301);
     }
     return NextResponse.next();
   }
   ```
