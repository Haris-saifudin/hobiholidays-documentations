# Area & Destination Hubs — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the **Area & Geography Domain**. Covers the **"Where To?" Search Autocomplete Widget** (`AreaAutocomplete`), debounced async searching, 3-tier hierarchical dropdown categorization (Continent/Country/City), and Destination Landing Hubs (`app/destinations/[slug]/page.tsx`).
>
> **Related Design Document:** [Area Domain Technical Design](../technical/area-technical-design.md)
> **API Contract:** [Area Contracts](../contracts/area-contract.md)
> **Backend Guide:** [Area Backend Guide](../backend/area-backend-guide.md)

---

## 🔍 "Where To?" Search Autocomplete Widget

The autocomplete widget provides instant discovery across Continents, Countries, and Cities as the user types in the homepage hero or global header:

```tsx
// components/search/area-autocomplete.tsx
'use client';

import { useState, useEffect, useRef } from 'react';
import { useRouter } from 'next/navigation';

export interface AreaSuggestion {
  id: string;
  name: string;
  slug: string;
  areaType: 'CONTINENT' | 'COUNTRY' | 'CITY';
  parentName?: string;
  grandparentName?: string;
  activePackagesCount: number;
}

export function AreaAutocomplete() {
  const router = useRouter();
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<AreaSuggestion[]>([]);
  const [isOpen, setIsOpen] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const wrapperRef = useRef<HTMLDivElement>(null);

  // Debounced fetch
  useEffect(() => {
    if (query.trim().length < 2) {
      setSuggestions([]);
      setIsOpen(false);
      return;
    }

    const timer = setTimeout(async () => {
      setIsLoading(true);
      try {
        const res = await fetch(
          `/api/v1/areas/autocomplete?q=${encodeURIComponent(query.trim())}`,
        );
        const json = await res.json();
        setSuggestions(json.data || []);
        setIsOpen(true);
      } catch (err) {
        console.error('Autocomplete error:', err);
      } finally {
        setIsLoading(false);
      }
    }, 250);

    return () => clearTimeout(timer);
  }, [query]);

  const handleSelect = (area: AreaSuggestion) => {
    setIsOpen(false);
    setQuery(area.name);
    if (area.areaType === 'CONTINENT') {
      router.push(`/tours?continent=${area.slug}`);
    } else if (area.areaType === 'COUNTRY') {
      router.push(`/tours?country=${area.slug}`);
    } else {
      router.push(`/destinations/${area.slug}`);
    }
  };

  return (
    <div ref={wrapperRef} className="relative w-full max-w-lg">
      <div className="relative">
        <input
          type="text"
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="Mau liburan ke mana? (e.g. Paris, Swiss, Eropa)"
          className="w-full px-5 py-3.5 pl-12 rounded-2xl border border-gray-200 shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 text-sm"
        />
        <span className="absolute left-4 top-3.5 text-gray-400">🔍</span>
        {isLoading && (
          <span className="absolute right-4 top-3.5 text-xs text-gray-400">Loading...</span>
        )}
      </div>

      {/* Autocomplete Dropdown List */}
      {isOpen && suggestions.length > 0 && (
        <ul className="absolute z-50 left-0 right-0 mt-2 bg-white rounded-2xl border border-gray-100 shadow-2xl overflow-hidden divide-y divide-gray-50 max-h-80 overflow-y-auto">
          {suggestions.map((item) => (
            <li
              key={item.id}
              onClick={() => handleSelect(item)}
              className="px-4 py-3 hover:bg-slate-50 cursor-pointer flex items-center justify-between transition"
            >
              <div>
                <p className="text-sm font-semibold text-gray-900">{item.name}</p>
                <p className="text-xs text-gray-500">
                  {item.areaType === 'CITY' && `${item.parentName}, ${item.grandparentName}`}
                  {item.areaType === 'COUNTRY' && `Negara di ${item.parentName}`}
                  {item.areaType === 'CONTINENT' && 'Benua'}
                </p>
              </div>
              <span className="text-xs px-2 py-0.5 rounded-md bg-blue-50 text-blue-700 font-medium">
                {item.activePackagesCount} Paket
              </span>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## 🏛️ Destination Landing Page Hub

```tsx
// app/destinations/[slug]/page.tsx
import { notFound } from 'next/navigation';
import { fetchApi } from '@/lib/api-client';
import { VariantCard } from '@/components/tour/variant-card';
import { BreadcrumbNav } from '@/components/ui/breadcrumb-nav';

interface Props {
  params: { slug: string };
}

export default async function DestinationPage({ params }: Props) {
  let destination: any;
  let packages: any[];

  try {
    destination = await fetchApi(`/areas/${params.slug}`);
    packages = await fetchApi(`/variants/search?country=${params.slug}&limit=12`);
  } catch (err) {
    notFound();
  }

  return (
    <main className="container mx-auto px-4 py-8">
      <BreadcrumbNav
        items={[
          { label: 'Home', href: '/' },
          { label: 'Destinations', href: '/destinations' },
          { label: destination.name, href: '' },
        ]}
      />

      {/* Destination Hero */}
      <div className="my-8">
        <h1 className="text-4xl font-extrabold">{destination.name} Tour Packages</h1>
        <p className="text-gray-600 mt-2 max-w-2xl">{destination.description}</p>
      </div>

      {/* Tours Grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {packages.map((v) => (
          <VariantCard key={v.id} variant={v} />
        ))}
      </div>
    </main>
  );
}
```
