# All Tours Catalog & Variant Cards — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the **All Tours** catalog page (`app/tours/page.tsx`) and the primary bookable unit: the **Variant Card Component** (`VariantCard`). Covers 2-tier Category badging, variant type visual badges (`STANDARD`, `SEASONAL`, `THEMED`, `PROMOTIONAL`), duration rendering, and Adult starting price formatting.
>
> **Related Design Document:** [Product Hierarchy Technical Design](../technical/product-hierarchy-technical-design.md)  
> **API Contract:** [Product Hierarchy Contracts](../contracts/product-hierarchy-contract.md)  
> **Backend Guide:** [Product Hierarchy Backend Guide](../backend/product-hierarchy-backend-guide.md)

---

## 🎨 Variant Type & Category Visual Badging System

The catalog surfaces distinct badging tokens based on `variant.variantType` and category taxonomy:

| Variant Type | Visual Role | Badge Colors (CSS / Tailwind) | Micro-Icon |
| :--- | :--- | :--- | :--- |
| **`STANDARD`** | Year-round core package | `bg-slate-100 text-slate-800 border-slate-300` | None / `⭐ Classic` |
| **`SEASONAL`** | Natural seasons (Spring/Autumn/Winter) | `bg-rose-50 text-rose-700 border-rose-200` | `🌸 Spring` / `🍂 Autumn` / `❄️ Winter` |
| **`THEMED`** | Festivals / Special interest | `bg-purple-50 text-purple-700 border-purple-200` | `🌷 Tulip` / `🎌 Festival` |
| **`PROMOTIONAL`** | Early bird / Flash sales | `bg-amber-50 text-amber-800 border-amber-300 font-bold animate-pulse` | `🔥 Flash Sale` / `⚡ Early Bird` |

---

## 🧩 Variant Card Component

```tsx
// components/tour/variant-card.tsx
import Image from 'next/image';
import Link from 'next/link';

export interface CategoryAssignmentDto {
  id: string;
  name: string;
  slug: string;
  dimensionCode?: string;
  dimensionName?: string;
}

export interface DestinationHierarchyDto {
  continent: string;
  continentSlug?: string;
  subContinent?: string | null;
  subContinentSlug?: string | null;
  country?: string | null;
  countrySlug?: string | null;
  poi?: string | null;
  poiSlug?: string | null;
}

export interface VariantBadge {
  id: string;
  code: string;
  label: string;
  backgroundColor: string;
  textColor: string;
  iconUrl?: string | null;
}

export interface VariantCardProps {
  variant: {
    id?: string;
    variantId?: string;
    code: string;
    name: string;
    slug: string;
    variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';
    badges?: VariantBadge[];
    categories?: CategoryAssignmentDto[];
    destinations?: DestinationHierarchyDto[];
    durationDays: number;
    durationNights: number;
    startingPrice: number;
    currency: string;
    coverImageUrl: string;
    nextDepartureDate?: string;
    product: {
      name: string;
      slug: string;
    };
  };
}

export function VariantCard({ variant }: VariantCardProps) {
  const getBadgeStyle = (type: string) => {
    switch (type) {
      case 'SEASONAL':
        return 'bg-rose-100 text-rose-800 border-rose-200';
      case 'THEMED':
        return 'bg-purple-100 text-purple-800 border-purple-200';
      case 'PROMOTIONAL':
        return 'bg-amber-100 text-amber-800 border-amber-300 font-bold';
      default:
        return 'bg-slate-100 text-slate-800 border-slate-200';
    }
  };

  const formattedPrice = new Intl.NumberFormat('id-ID', {
    style: 'currency',
    currency: variant.currency || 'IDR',
    maximumFractionDigits: 0,
  }).format(variant.startingPrice);

  return (
    <article className="group bg-white rounded-2xl border border-gray-100 overflow-hidden shadow-sm hover:shadow-xl transition-all duration-300 flex flex-col">
      {/* 1. Image Container */}
      <div className="relative aspect-[16/10] w-full overflow-hidden bg-slate-100">
        <Image
          src={variant.coverImageUrl}
          alt={variant.name}
          fill
          sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
          className="object-cover group-hover:scale-105 transition-transform duration-500 ease-out"
          priority={false}
        />

        {/* Top Badges: Promotional Badges + Variant Type + Category */}
        <div className="absolute top-3 left-3 flex flex-wrap gap-1.5">
          {variant.badges && variant.badges.map((badge) => (
            <span
              key={badge.id}
              style={{ backgroundColor: badge.backgroundColor, color: badge.textColor }}
              className="px-2.5 py-1 rounded-full text-xs font-semibold shadow-sm backdrop-blur-md border border-black/5"
            >
              {badge.label}
            </span>
          ))}
          <span className={`px-2.5 py-1 rounded-full text-xs border font-medium shadow-sm backdrop-blur-md ${getBadgeStyle(variant.variantType)}`}>
            {variant.variantType}
          </span>
          {variant.categories?.slice(0, 1).map((cat) => (
            <span key={cat.id} className="px-2.5 py-1 rounded-full text-xs font-medium bg-white/90 text-slate-700 shadow-sm backdrop-blur-md">
              {cat.name}
            </span>
          ))}
        </div>

        {/* Duration Badge */}
        <div className="absolute bottom-3 right-3 bg-black/70 backdrop-blur-md text-white text-xs px-2.5 py-1 rounded-lg font-medium">
          {variant.durationDays}D / {variant.durationNights}N
        </div>
      </div>

      {/* 2. Card Body */}
      <div className="p-5 flex-1 flex flex-col justify-between">
        <div>
          <div className="flex items-center gap-2">
            <span className="text-xs text-gray-500 uppercase tracking-wider font-semibold">
              {variant.product.name}
            </span>
            {variant.categories && variant.categories.length > 1 && (
              <span className="text-[10px] px-1.5 py-0.5 rounded bg-gray-100 text-gray-600">
                {variant.categories[1].name}
              </span>
            )}
          </div>
          <h3 className="text-lg font-bold text-gray-900 mt-1 line-clamp-2 group-hover:text-blue-600 transition-colors">
            {variant.name}
          </h3>

          {/* Destination Markers (Supports POI, Country, Sub-Continent, or Continent Anchoring) */}
          {variant.destinations && variant.destinations.length > 0 && (
            <div className="flex flex-wrap gap-1.5 mt-2.5">
              {variant.destinations.slice(0, 3).map((d, idx) => (
                <span
                  key={idx}
                  className="text-[11px] font-medium text-slate-600 bg-slate-100 px-2 py-0.5 rounded-md flex items-center gap-1"
                >
                  📍 {d.poi || d.country || d.subContinent || d.continent}
                </span>
              ))}
              {variant.destinations.length > 3 && (
                <span className="text-[10px] text-slate-400 self-center">
                  +{variant.destinations.length - 3} lainnya
                </span>
              )}
            </div>
          )}
        </div>

        {/* 3. Pricing & Call to Action */}
        <div className="pt-4 mt-4 border-t border-gray-50 flex items-end justify-between">
          <div>
            <p className="text-xs text-gray-500">Mulai dari (Dewasa)</p>
            <p className="text-lg font-extrabold text-blue-600">
              {variant.startingPrice > 0 ? formattedPrice : 'Hubungi Kami'}
            </p>
          </div>

          <Link
            href={`/tours/${variant.product.slug}/${variant.slug}`}
            className="px-4 py-2 bg-slate-900 hover:bg-blue-600 text-white rounded-xl text-xs font-semibold transition"
          >
            Lihat Detail
          </Link>
        </div>
      </div>
    </article>
  );
}
```
