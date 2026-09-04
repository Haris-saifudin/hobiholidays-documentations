# Dynamic SEO & Rich Snippets — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for Search Engine Optimization (SEO), Social Sharing (OpenGraph / Twitter), and Google Rich Results (Schema.org JSON-LD) across the Hobiholidays web application. Covers Next.js App Router `generateMetadata()`, Schema.org injection, dynamic XML sitemaps (`app/sitemap.ts`), and crawler directives (`app/robots.ts`).
>
> **Related Design Document:** [SEO Technical Design](../technical/seo-technical-design.md)
> **API Contract:** [SEO Contracts](../contracts/seo-contract.md)
> **Backend Guide:** [SEO Backend Guide](../backend/seo-backend-guide.md)

---

## ⚡ Dynamic Metadata (`generateMetadata`)

In Next.js App Router, metadata is generated server-side before HTML streaming begins:

```tsx
// app/tours/[productSlug]/[variantSlug]/page.tsx
import type { Metadata } from 'next';

interface Props {
  params: { productSlug: string; variantSlug: string };
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const res = await fetch(
    `https://api.hobiholidays.com/api/v1/variants/${params.variantSlug}`,
    { next: { revalidate: 3600 } }, // Revalidate every hour (ISR)
  );

  if (!res.ok) {
    return {
      title: 'Tour Not Found | Hobiholidays',
      robots: { index: false, follow: false },
    };
  }

  const { data: variant } = await res.json();

  const title =
    variant.seo?.metaTitle ||
    `${variant.name} (${variant.durationDays}D/${variant.durationNights}N) | Hobiholidays`;

  const description =
    variant.seo?.metaDescription ||
    `Ikuti tour ${variant.name} (${variant.durationDays} Hari). Mulai IDR ${new Intl.NumberFormat('id-ID').format(variant.startingPrice)}. Booking aman dan terpercaya bersama Hobiholidays.`;

  const canonicalUrl =
    variant.seo?.canonicalUrl ||
    `https://www.hobiholidays.com/tours/${params.productSlug}/${params.variantSlug}`;

  const ogImage =
    variant.seo?.ogImageUrl ||
    variant.coverImageUrl ||
    'https://cdn.hobiholidays.com/defaults/og-tour-default.jpg';

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

## 🏷️ Schema.org Structured Data Component (`TouristTrip` + `Offer`)

```tsx
// components/seo/jsonld-script.tsx
export function JsonLdScript({ variant }: { variant: any }) {
  const schema = {
    '@context': 'https://schema.org',
    '@type': 'TouristTrip',
    name: variant.name,
    description: variant.description || `Paket Tour ${variant.name}`,
    touristType: ['Family', 'Group', 'Leisure'],
    offers: (variant.trips || []).map((t: any) => ({
      '@type': 'Offer',
      price: t.sellingPrice,
      priceCurrency: 'IDR',
      availability:
        t.availableSeats > 0
          ? 'https://schema.org/InStock'
          : 'https://schema.org/SoldOut',
      validFrom: new Date().toISOString().split('T')[0],
      url: `https://www.hobiholidays.com/tours/${variant.productSlug}/${variant.slug}`,
    })),
  };

  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(schema) }}
    />
  );
}
```

---

## 🗺️ Dynamic XML Sitemap Generator (`app/sitemap.ts`)

```typescript
// app/sitemap.ts
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = 'https://www.hobiholidays.com';

  // 1. Fetch active variants
  const variantsRes = await fetch('https://api.hobiholidays.com/api/v1/variants?limit=500', {
    next: { revalidate: 86400 },
  });
  const { data: variants } = await variantsRes.json();

  const tourUrls: MetadataRoute.Sitemap = variants.map((v: any) => ({
    url: `${baseUrl}/tours/${v.productSlug}/${v.slug}`,
    lastModified: v.updatedAt || new Date().toISOString(),
    changeFrequency: 'weekly',
    priority: 0.9,
  }));

  // 2. Fetch destination areas
  const areasRes = await fetch('https://api.hobiholidays.com/api/v1/areas/tree', {
    next: { revalidate: 86400 },
  });
  const { data: areas } = await areasRes.json();

  const areaUrls: MetadataRoute.Sitemap = areas.map((a: any) => ({
    url: `${baseUrl}/destinations/${a.slug}`,
    lastModified: new Date().toISOString(),
    changeFrequency: 'monthly',
    priority: 0.8,
  }));

  return [
    {
      url: baseUrl,
      lastModified: new Date().toISOString(),
      changeFrequency: 'daily',
      priority: 1.0,
    },
    {
      url: `${baseUrl}/tours`,
      lastModified: new Date().toISOString(),
      changeFrequency: 'daily',
      priority: 0.9,
    },
    ...tourUrls,
    ...areaUrls,
  ];
}
```

---

## 🤖 Dynamic Crawler Directives (`app/robots.ts`)

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
          '/checkout/',
          '/account/',
          '/admin/',
          '/*?*search=', // Disallow search results with parameters to prevent crawl budget waste
        ],
      },
      {
        userAgent: 'Googlebot',
        allow: '/',
        disallow: ['/api/', '/checkout/'],
      },
    ],
    sitemap: 'https://www.hobiholidays.com/sitemap.xml',
  };
}
```
