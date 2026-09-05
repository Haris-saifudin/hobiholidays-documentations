# Product Detail Page (PDP) — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the Variant Detail Page (PDP) located at `app/tours/[productSlug]/[variantSlug]/page.tsx`. Demonstrates asynchronous tabbed UI rendering, Variant default itinerary with Trip override badges, Age-band pricing selector with itemized inclusion breakdown drawers, Add-on selectors, and official itinerary PDF brochure downloader.
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
│ Title: Grand West Europe Spring 2026        │ Price: IDR 28.000.000     │
│ Category: Classic Series (Tour Series)      │ [Age Band: Adult / Infant] │
│ Duration: 11 Days / 9 Nights                │ [Rincian Komponen Biaya ↗]│
│ Code: GWE-SPR-2026   [🌸 Spring Edition]    │ Departure Date Selector   │
│                                             │ Quota: 8 Seats Remaining  │
│                                             │ [Add-on Extras Selector]  │
│                                             │ [Book This Tour Button]   │
│                                             │ [📄 Download PDF Brochure]│
├─────────────────────────────────────────────┴───────────────────────────┤
│ [Tab Navigation: Itinerary (Master / Override) | Locations | Inclusions]│
│                                                                         │
│ <Suspense fallback={<ItinerarySkeleton />}>                             │
│   <DayByDayItineraryList variantId={variant.id} tripId={selectedTripId} │
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
          { label: variant.product.name, href: `/tours/${params.productSlug}` },
          { label: variant.name, href: '' },
        ]}
      />

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8 mt-6">
        {/* Left Column: Media & Core Info (2 cols) */}
        <div className="lg:col-span-2 space-y-6">
          <GalleryCarousel productId={variant.product.id} />

          <div className="border-b pb-4">
            <div className="flex items-center gap-3">
              <h1 className="text-3xl font-bold">{variant.name}</h1>
              <span className="px-3 py-1 rounded-full text-xs font-semibold bg-emerald-100 text-emerald-800">
                {variant.variantType}
              </span>
              {variant.product.category && (
                <span className="px-3 py-1 rounded-full text-xs font-medium bg-blue-50 text-blue-700">
                  {variant.product.category.name}
                </span>
              )}
            </div>
            <p className="text-gray-600 mt-2 text-lg">
              {variant.durationDays} Hari / {variant.durationNights} Malam
            </p>
          </div>

          {/* Tabbed Content with Streaming Suspense */}
          <ItineraryTabs
            variantId={variant.id}
            defaultItinerary={variant.itinerary}
          />
        </div>

        {/* Right Column: Sticky Booking, Pricing Breakdown & Add-ons (1 col) */}
        <div className="space-y-6">
          <BookingCard
            variant={variant}
            trips={variant.trips}
            addons={variant.addons}
          />

          {(variant.itineraryPdfUrl || variant.product.itineraryPdfUrl) && (
            <BrochureDownloadButton
              url={variant.itineraryPdfUrl || variant.product.itineraryPdfUrl}
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

## 💰 All-Inclusive Pricing Drawer & Inclusions

Allows travelers to view the all-inclusive package pricing tier details (e.g. Adult GWE Summer) along with quota allocation and supplementary package inclusions:

```tsx
// components/tour/pricing-breakdown-drawer.tsx
'use client';

import { useState } from 'react';

export interface SupplementaryInclusion {
  category: 'INCLUDED' | 'EXCLUDED' | 'IMPORTANT_INFO';
  content: string;
}

export interface PricingComponentItem {
  id: string;
  name: string;
  description?: string;
  amount?: number;
  isIncluded: boolean;
  sortOrder?: number;
}

export interface PricingTier {
  id: string;
  ageBand: 'ADULT' | 'INFANT';
  basePrice: number;
  sellingPrice: number;
  consumesQuota: boolean;
  components?: PricingComponentItem[];
}

interface Props {
  pricing: PricingTier;
  inclusions?: SupplementaryInclusion[];
}

export function PricingBreakdownDrawer({ pricing, inclusions = [] }: Props) {
  const [isOpen, setIsOpen] = useState(false);

  const formatIDR = (amt: number) =>
    new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(amt);

  return (
    <div>
      <button
        onClick={() => setIsOpen(true)}
        className="text-xs text-blue-600 hover:underline font-medium flex items-center gap-1"
      >
        Lihat Rincian Biaya Paket ↗
      </button>

      {isOpen && (
        <div className="fixed inset-0 z-50 bg-black/50 flex justify-end">
          <div className="w-full max-w-md bg-white h-full p-6 overflow-y-auto shadow-2xl flex flex-col justify-between">
            <div>
              <div className="flex items-center justify-between border-b pb-4">
                <h3 className="text-lg font-bold text-gray-900">
                  Rincian Biaya: {pricing.ageBand === 'ADULT' ? 'Dewasa' : 'Bayi / Infant'}
                </h3>
                <button onClick={() => setIsOpen(false)} className="text-gray-400 hover:text-gray-600 text-xl font-bold">
                  ✕
                </button>
              </div>

              <div className="mt-3 p-2.5 rounded-lg text-xs font-medium bg-slate-50 text-slate-700">
                {pricing.consumesQuota ? (
                  <span>✓ Mengalokasikan 1 kursi/kuota peserta grup</span>
                ) : (
                  <span className="text-emerald-700">✓ Lap infant (tidak mengalokasikan kuota kursi terpisah)</span>
                )}
              </div>

              {/* Itemized Cost Breakdown Components */}
              {pricing.components && pricing.components.length > 0 && (
                <div className="mt-4 space-y-2">
                  <p className="text-xs font-semibold text-gray-500 uppercase">Komponen & Rincian Fasilitas</p>
                  <div className="space-y-2">
                    {pricing.components.map((comp) => (
                      <div key={comp.id} className="p-3 rounded-xl border border-gray-100 bg-slate-50 flex items-start justify-between gap-3">
                        <div className="space-y-0.5">
                          <p className="text-sm font-medium text-gray-900 flex items-center gap-1.5">
                            <span className="text-emerald-600 font-bold">✓</span>
                            {comp.name}
                          </p>
                          {comp.description && (
                            <p className="text-xs text-gray-500 pl-4">{comp.description}</p>
                          )}
                        </div>
                        {comp.amount && (
                          <span className="text-xs font-semibold text-slate-600 shrink-0">
                            {formatIDR(comp.amount)}
                          </span>
                        )}
                      </div>
                    ))}
                  </div>
                </div>
              )}

              {/* General Inclusions */}
              {inclusions.some((inc) => inc.category === 'INCLUDED') && (
                <div className="mt-4 space-y-3">
                  <p className="text-xs font-semibold text-gray-500 uppercase">Fasilitas Termasuk Tambahan</p>
                  {inclusions.filter((inc) => inc.category === 'INCLUDED').map((inc, idx) => (
                    <div key={idx} className="p-3 rounded-xl border border-gray-100 bg-slate-50 flex items-start gap-2">
                      <span className="text-emerald-600 font-bold">✓</span>
                      <p className="text-sm text-gray-800">{inc.content}</p>
                    </div>
                  ))}
                </div>
              )}
            </div>

            <div className="border-t pt-4 mt-6">
              <div className="flex justify-between items-center text-base font-bold">
                <span>Total Harga Paket All-Inclusive</span>
                <span className="text-blue-600 text-xl">{formatIDR(pricing.sellingPrice)}</span>
              </div>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
```

---

## 🧩 Add-on Extra Options Selector

Add-ons are optional extras added on top of the full package base price:

```tsx
// components/tour/addon-selector.tsx
'use client';

export interface AddonItem {
  id: string;
  code?: string;
  name: string;
  description?: string;
  addonType?: string;
  chargeType?: 'PER_PAX' | 'PER_ROOM' | 'PER_BOOKING';
  price: number;
  currency?: string;
  applicableAgeBand?: 'ADULT' | 'INFANT' | null;
  isMandatory?: boolean;
  maxQuantity?: number;
}

interface Props {
  addons: AddonItem[];
  selectedAddons: Record<string, number>;
  onChange: (addonId: string, qty: number) => void;
}

export function AddonSelector({ addons, selectedAddons, onChange }: Props) {
  if (!addons || addons.length === 0) return null;

  return (
    <div className="space-y-3 pt-4 border-t">
      <h4 className="text-sm font-bold text-gray-900">Tambahan Opsional (Add-on)</h4>
      {addons.map((addon) => {
        const qty = selectedAddons[addon.id] || 0;
        const unitLabel = addon.chargeType === 'PER_ROOM' ? 'kamar' : 'orang';
        return (
          <div key={addon.id} className="p-3 rounded-xl border border-gray-100 flex items-center justify-between">
            <div>
              <p className="text-sm font-semibold text-gray-800">{addon.name}</p>
              <p className="text-xs text-gray-500">
                +Rp {addon.price.toLocaleString('id-ID')} / {unitLabel}
                {addon.applicableAgeBand && ` (${addon.applicableAgeBand === 'ADULT' ? 'Dewasa' : 'Bayi'})`}
              </p>
            </div>
            <input
              type="checkbox"
              checked={qty > 0}
              onChange={(e) => onChange(addon.id, e.target.checked ? 1 : 0)}
              className="h-4 w-4 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
            />
          </div>
        );
      })}
    </div>
  );
}
```

---

## 📄 Official Itinerary PDF Brochure Component

> Itinerary PDF brochures are compiled externally by **ATW**. Hobiholidays does not generate PDFs internally; the application provides direct download access via the external ATW URL (`itineraryPdfUrl`).

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
        <p className="text-xs text-gray-500">Format PDF • Disusun resmi oleh ATW</p>
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
