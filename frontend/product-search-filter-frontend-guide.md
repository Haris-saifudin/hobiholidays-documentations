# Catalog Search & Filter UI — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the **All Tours Search & Filter Engine** (`app/tours/page.tsx`). Demonstrates bidirectional URL query string synchronization using Next.js `useSearchParams` and `useRouter`, multi-criteria filter sidebar, active filter chips, skeleton loaders, and empty state fallbacks.
>
> **Related Design Document:** [Search & Filter Architecture](../technical/product-search-filter-technical-design.md)
> **API Contract:** [Search & Filter Contracts](../contracts/product-search-filter-contract.md)
> **Backend Guide:** [Search & Filter Backend Guide](../backend/product-search-filter-backend-guide.md)

---

## 🏗️ Search & Filter Layout Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Search Bar: [ 🔍 Search by tour or city...               ] [Search Btn] │
├──────────────────────────┬──────────────────────────────────────────────┤
│ 🎛️ Filters Sidebar       │ Results: 14 Tour Packages Found              │
│                          │ Active Chips: [Eropa ✕] [Budget <50M ✕]     │
│ • Continent & Country    ├──────────────────────────────────────────────┤
│ • Budget Range Slider    │ [VariantCard]   [VariantCard]   [VariantCard]│
│ • Total Pack (1-20 Pax)  │ [VariantCard]   [VariantCard]   [VariantCard]│
│ • Departure Month        ├──────────────────────────────────────────────┤
│ • Variant Type Pills     │ Pagination: [ < Prev ] [ 1 ] [ 2 ] [ Next > ]│
└──────────────────────────┴──────────────────────────────────────────────┘
```

---

## 🔄 Bidirectional URL Query State Hook

Filters synchronize directly to the URL query string, making all search states shareable and bookmarkable:

```tsx
// hooks/use-search-filters.ts
'use client';

import { useRouter, useSearchParams, usePathname } from 'next/navigation';
import { useTransition, useCallback } from 'react';

export function useSearchFilters() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [isPending, startTransition] = useTransition();

  const setFilter = useCallback(
    (key: string, value: string | null) => {
      const params = new URLSearchParams(searchParams.toString());
      if (value === null || value === '' || value === 'ALL') {
        params.delete(key);
      } else {
        params.set(key, value);
      }
      params.delete('page'); // Reset to page 1 on filter change

      startTransition(() => {
        router.push(`${pathname}?${params.toString()}`);
      });
    },
    [router, pathname, searchParams],
  );

  const clearAllFilters = useCallback(() => {
    startTransition(() => {
      router.push(pathname);
    });
  }, [router, pathname]);

  return {
    filters: {
      continent: searchParams.get('continent') || '',
      country: searchParams.get('country') || '',
      name: searchParams.get('name') || '',
      minPrice: searchParams.get('minPrice') || '',
      maxPrice: searchParams.get('maxPrice') || '',
      totalPack: searchParams.get('totalPack') || '',
      departureMonth: searchParams.get('departureMonth') || '',
      variantType: searchParams.get('variantType') || '',
      page: searchParams.get('page') || '1',
    },
    setFilter,
    clearAllFilters,
    isPending,
  };
}
```

---

## 🎛️ Filter Sidebar Component

```tsx
// components/search/filter-sidebar.tsx
'use client';

import { useSearchFilters } from '@/hooks/use-search-filters';

export function FilterSidebar() {
  const { filters, setFilter, clearAllFilters } = useSearchFilters();

  return (
    <aside className="w-full lg:w-72 space-y-6 bg-white p-6 rounded-2xl border border-gray-100 shadow-sm">
      <div className="flex items-center justify-between pb-4 border-b border-gray-100">
        <h3 className="font-bold text-gray-900">Filter Pencarian</h3>
        <button
          onClick={clearAllFilters}
          className="text-xs text-blue-600 hover:underline font-semibold"
        >
          Reset
        </button>
      </div>

      {/* 1. Variant Type */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Tipe Paket
        </label>
        <div className="space-y-2">
          {['ALL', 'STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL'].map((type) => (
            <label key={type} className="flex items-center gap-2 cursor-pointer text-sm">
              <input
                type="radio"
                name="variantType"
                value={type}
                checked={filters.variantType === type || (type === 'ALL' && !filters.variantType)}
                onChange={() => setFilter('variantType', type === 'ALL' ? null : type)}
                className="text-blue-600"
              />
              <span>{type === 'ALL' ? 'Semua Tipe' : type}</span>
            </label>
          ))}
        </div>
      </div>

      {/* 2. Total Pack / Pax Capacity */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Jumlah Peserta (Pax)
        </label>
        <input
          type="number"
          min="1"
          max="20"
          value={filters.totalPack}
          onChange={(e) => setFilter('totalPack', e.target.value || null)}
          placeholder="e.g. 2 Orang"
          className="w-full px-4 py-2 border rounded-xl text-sm"
        />
      </div>

      {/* 3. Departure Month */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Bulan Keberangkatan
        </label>
        <input
          type="month"
          value={filters.departureMonth}
          onChange={(e) => setFilter('departureMonth', e.target.value || null)}
          className="w-full px-4 py-2 border rounded-xl text-sm"
        />
      </div>
    </aside>
  );
}
```
