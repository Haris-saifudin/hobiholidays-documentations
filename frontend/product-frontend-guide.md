# Product Detail Page (PDP) — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the Variant Detail Page (PDP) located at `app/tours/[productSlug]/[variantSlug]/page.tsx`. Demonstrates asynchronous tabbed UI rendering, split sub-resource data fetching with React Suspense, official itinerary PDF brochure downloader, and breadcrumbs.
>
> **Related Design Document:** [Product Technical Design](../technical/product-technical-design.md)
> **API Contract:** [Product Contracts](../contracts/product-contract.md)
> **Backend Guide:** [Product Backend Guide](../backend/product-backend-guide.md)

---

## 🏗️ Page Layout & Architecture

The Variant Detail Page (PDP) presents the master brand narrative alongside variant-specific departure dates:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Breadcrumbs: Home > Tours > Europe > Grand West Europe Spring 2026      │
├─────────────────────────────────────────────────────────────────────────┤
│ [Hero Image & Gallery Carousel]             │ [Sticky Booking Card]     │
│ Title: Grand West Europe Spring 2026        │ Price: IDR 34.500.000     │
│ Duration: 11 Days / 10 Nights               │ Departure Date Selector   │
│ Code: GWE-SPR-2026   [🌸 Spring Edition]    │ Quota: 8 Seats Remaining  │
│                                             │ [Book This Tour Button]   │
│                                             │ [📄 Download PDF Brochure]│
├─────────────────────────────────────────────┴───────────────────────────┤
│ [Tab Navigation: Overview | Itinerary | Locations | Inclusions/Exclusions]│
│                                                                         │
│ <Suspense fallback={<ItinerarySkeleton />}>                             │
│   <DayByDayItineraryList productId={variant.productId} />               │
│ </Suspense>                                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ⚡ Server Component Implementation

```tsx
// app/tours/[productSlug]/[variantSlug]/page.tsx
import { Suspense } from 'react';
import { notFound } from 'next/navigation';
import { fetchApi } from '@/lib/api-client';
import { BookingCard } from '@/components/tour/booking-card';
import { GalleryCarousel } from '@/components/media/gallery-carousel';
import { ItineraryTabs } from '@/components/tour/itinerary-tabs';
import { BrochureDownloadButton } from '@/components/media/brochure-download-button';
import { BreadcrumbNav } from '@/components/ui/breadcrumb-nav';
import { JsonLdScript } from '@/components/seo/jsonld-script';

interface Props {
  params: { productSlug: string; variantSlug: string };
}

export default async function VariantDetailPage({ params }: Props) {
  // 1. Fetch primary variant details
  let variant: any;
  try {
    variant = await fetchApi(`/variants/${params.variantSlug}`, {
      next: { revalidate: 3600, tags: [`variant-${params.variantSlug}`] },
    });
  } catch (err) {
    notFound();
  }

  return (
    <main className="container mx-auto px-4 py-8">
      {/* Schema.org TouristTrip JSON-LD */}
      <JsonLdScript variant={variant} />

      {/* Breadcrumb Navigation */}
      <BreadcrumbNav
        items={[
          { label: 'Home', href: '/' },
          { label: 'Tours', href: '/tours' },
          { label: variant.productName, href: `/tours/${params.productSlug}` },
          { label: variant.name, href: '' },
        ]}
      />

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8 mt-6">
        {/* Left Column: Media & Core Info (2 cols) */}
        <div className="lg:col-span-2 space-y-6">
          <GalleryCarousel productId={variant.productId} />

          <div className="border-b pb-4">
            <div className="flex items-center gap-3">
              <h1 className="text-3xl font-bold">{variant.name}</h1>
              <span className="px-3 py-1 rounded-full text-xs font-semibold bg-emerald-100 text-emerald-800">
                {variant.variantType}
              </span>
            </div>
            <p className="text-gray-600 mt-2 text-lg">
              {variant.durationDays} Hari / {variant.durationNights} Malam
            </p>
          </div>

          {/* Tabbed Content with Streaming Suspense */}
          <ItineraryTabs productId={variant.productId} />
        </div>

        {/* Right Column: Sticky Booking & Brochure Download (1 col) */}
        <div className="space-y-6">
          <BookingCard variant={variant} trips={variant.trips} />

          {variant.itineraryPdfUrl && (
            <BrochureDownloadButton
              url={variant.itineraryPdfUrl}
              tourName={variant.name}
            />
          )}
        </div>
      </div>
    </main>
  );
}
```

---

## 📄 Official Itinerary PDF Brochure Component

```tsx
// components/media/brochure-download-button.tsx
'use client';

import { useState } from 'react';

interface Props {
  url: string;
  tourName: string;
}

export function BrochureDownloadButton({ url, tourName }: Props) {
  const [isDownloading, setIsDownloading] = useState(false);

  const handleDownload = async () => {
    setIsDownloading(true);
    try {
      const response = await fetch(url);
      const blob = await response.blob();
      const blobUrl = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = blobUrl;
      a.download = `${tourName.replace(/[^a-zA-Z0-9]/g, '_')}_Brochure.pdf`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      window.URL.revokeObjectURL(blobUrl);
    } catch (error) {
      console.error('Download failed:', error);
      window.open(url, '_blank');
    } finally {
      setIsDownloading(false);
    }
  };

  return (
    <div className="p-4 border rounded-xl bg-slate-50 flex items-center justify-between">
      <div>
        <h4 className="font-semibold text-sm">Download Itinerary Resmi</h4>
        <p className="text-xs text-gray-500">Format PDF • Lengkap dengan Jadwal & Ketentuan</p>
      </div>
      <button
        onClick={handleDownload}
        disabled={isDownloading}
        className="px-4 py-2 bg-blue-600 hover:bg-blue-700 text-white rounded-lg text-sm font-medium transition flex items-center gap-2"
      >
        {isDownloading ? 'Mengunduh...' : 'Unduh PDF'}
      </button>
    </div>
  );
}
```
