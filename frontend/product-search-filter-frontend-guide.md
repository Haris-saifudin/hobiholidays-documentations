# Catalog Search & Filter UI — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the **All Tours Search & Filter Engine** (`app/tours/page.tsx`). Demonstrates bidirectional URL query string synchronization using Next.js `useSearchParams` and `useRouter`, multi-criteria filter sidebar supporting 2-tier Categories and 4-tier Area selectors, active filter chips, skeleton loaders, and empty state fallbacks.
>
> **Related Design Document:** [Search & Filter Architecture](../technical/product-search-filter-technical-design.md)  
> **API Contract:** [Search & Filter Contracts](../contracts/product-search-filter-contract.md)  
> **Backend Guide:** [Search & Filter Backend Guide](../backend/product-search-filter-backend-guide.md)

---

## 🏗️ Search & Filter Layout Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Search Bar: [ 🔍 Search by tour, POI, or country...     ] [Search Btn]  │
├──────────────────────────┬──────────────────────────────────────────────┤
│ 🎛️ Filters Sidebar       │ Results: 14 Tour Packages Found              │
│                          │ Active Chips: [Eropa ✕] [Classic Series ✕]   │
│ • Category (2-tier)      ├──────────────────────────────────────────────┤
│ • Destination (4-tier)   │ [VariantCard]   [VariantCard]   [VariantCard]│
│ • Budget Range Slider    │ [VariantCard]   [VariantCard]   [VariantCard]│
│ • Total Pack (1-20 Pax)  ├──────────────────────────────────────────────┤
│ • Departure Month        │ Pagination: [ < Prev ] [ 1 ] [ 2 ] [ Next > ]│
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
      parentCategorySlug: searchParams.get('parentCategorySlug') || '',
      categorySlug: searchParams.get('categorySlug') || '',
      continentSlug: searchParams.get('continentSlug') || '',
      subContinentSlug: searchParams.get('subContinentSlug') || '',
      countrySlug: searchParams.get('countrySlug') || '',
      poiSlug: searchParams.get('poiSlug') || '',
      productName: searchParams.get('productName') || '',
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

## 🎛️ Filter Master Data Hook & Dynamic Sidebar Component

### 1. Dynamic Master Data Hook (`use-filter-options.ts`)

```tsx
// hooks/use-filter-options.ts
'use client';

import { useQuery } from '@tanstack/react-query';
import { FilterOptionsResponseDto } from '@/types/search-filter';

async function fetchFilterOptions(): Promise<FilterOptionsResponseDto> {
  const res = await fetch('/api/v1/variants/search/filter-options', {
    headers: { 'Accept': 'application/json' },
    next: { revalidate: 300 }, // 5 minutes client revalidation
  });
  if (!res.ok) throw new Error('Failed to load search filter options');
  const json = await res.json();
  return json.data;
}

export function useFilterOptions() {
  return useQuery({
    queryKey: ['catalog-filter-options'],
    queryFn: fetchFilterOptions,
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 30 * 60 * 1000,
  });
}
```

---

### 2. Dynamic Filter Sidebar Component (`filter-sidebar.tsx`)

```tsx
// components/search/filter-sidebar.tsx
'use client';

import { useSearchFilters } from '@/hooks/use-search-filters';
import { useFilterOptions } from '@/hooks/use-filter-options';

export function FilterSidebar() {
  const { filters, setFilter, clearAllFilters } = useSearchFilters();
  const { data: options, isLoading } = useFilterOptions();

  if (isLoading) {
    return (
      <aside className="w-full lg:w-72 space-y-6 bg-white p-6 rounded-2xl border border-gray-100 shadow-sm animate-pulse">
        <div className="h-6 bg-gray-200 rounded w-1/2 mb-4" />
        <div className="space-y-4">
          <div className="h-10 bg-gray-100 rounded-xl" />
          <div className="h-10 bg-gray-100 rounded-xl" />
          <div className="h-10 bg-gray-100 rounded-xl" />
        </div>
      </aside>
    );
  }

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

      {/* 1. Category Filter (2-tier taxonomy) */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Kategori Paket
        </label>
        <select
          value={filters.categorySlug}
          onChange={(e) => setFilter('categorySlug', e.target.value || null)}
          className="w-full px-3.5 py-2.5 rounded-xl border border-gray-200 text-sm focus:ring-2 focus:ring-blue-500 outline-none"
        >
          <option value="">Semua Kategori</option>
          {options?.categories.map((parent) => (
            <optgroup key={parent.id} label={parent.name}>
              {parent.children.map((child) => (
                <option key={child.id} value={child.slug}>
                  {child.name} ({child.activePackagesCount})
                </option>
              ))}
            </optgroup>
          ))}
        </select>
      </div>

      {/* 2. Destination Filter (4-tier Area hierarchy) */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Destinasi Tour
        </label>
        <select
          value={filters.countrySlug || filters.continentSlug}
          onChange={(e) => {
            const val = e.target.value;
            if (!val) {
              setFilter('continentSlug', null);
              setFilter('countrySlug', null);
            } else if (val.startsWith('cont:')) {
              setFilter('continentSlug', val.replace('cont:', ''));
              setFilter('countrySlug', null);
            } else if (val.startsWith('co:')) {
              setFilter('countrySlug', val.replace('co:', ''));
              setFilter('continentSlug', null);
            }
          }}
          className="w-full px-3.5 py-2.5 rounded-xl border border-gray-200 text-sm focus:ring-2 focus:ring-blue-500 outline-none"
        >
          <option value="">Semua Destinasi</option>
          {options?.destinations.map((cont) => (
            <optgroup key={cont.id} label={`Benua: ${cont.name} (${cont.activePackagesCount})`}>
              <option value={`cont:${cont.slug}`}>Seluruh {cont.name}</option>
              {cont.countries.map((country) => (
                <option key={country.id} value={`co:${country.slug}`}>
                  — {country.name} ({country.activePackagesCount} Paket)
                </option>
              ))}
            </optgroup>
          ))}
        </select>
      </div>

      {/* 3. Variant Type */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Tipe Paket
        </label>
        <div className="space-y-2">
          <label className="flex items-center gap-2 cursor-pointer text-sm">
            <input
              type="radio"
              name="variantType"
              value="ALL"
              checked={!filters.variantType}
              onChange={() => setFilter('variantType', null)}
              className="text-blue-600"
            />
            <span>Semua Tipe</span>
          </label>
          {options?.variantTypes.map((vt) => (
            <label key={vt.key} className="flex items-center gap-2 cursor-pointer text-sm">
              <input
                type="radio"
                name="variantType"
                value={vt.key}
                checked={filters.variantType === vt.key}
                onChange={() => setFilter('variantType', vt.key)}
                className="text-blue-600"
              />
              <span>
                {vt.label} <span className="text-xs text-gray-400">({vt.count})</span>
              </span>
            </label>
          ))}
        </div>
      </div>

      {/* 4. Departure Month Filter */}
      <div>
        <label className="text-xs font-bold uppercase tracking-wider text-gray-500 block mb-3">
          Bulan Keberangkatan
        </label>
        <select
          value={filters.departureMonth}
          onChange={(e) => setFilter('departureMonth', e.target.value || null)}
          className="w-full px-3.5 py-2.5 rounded-xl border border-gray-200 text-sm focus:ring-2 focus:ring-blue-500 outline-none"
        >
          <option value="">Semua Jadwal</option>
          {options?.departureMonths.map((m) => (
            <option key={m.value} value={m.value}>
              {m.label} ({m.activeTripsCount} Jadwal)
            </option>
          ))}
        </select>
      </div>

      {/* 5. Total Pack / Pax Capacity */}
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
    </aside>
  );
}
```
