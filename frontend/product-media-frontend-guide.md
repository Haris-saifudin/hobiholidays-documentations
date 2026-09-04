# Media Assets & Next/Image Optimization — Next.js Frontend Implementation Guide

> **Pillar 4: Next.js Frontend Implementation**
> Frontend implementation guide for the **Product Media Subsystem**. Covers complete **Next.js Image Component (`next/image`)** integration, `next.config.ts` remote patterns for **Phase 1 (Database Stream API)** vs **Phase 2 (Cloudflare Edge CDN)**, responsive image presets, and interactive gallery carousels.
>
> **Related Design Document:** [Product Media Technical Design](../technical/product-media-technical-design.md)
> **API Contract:** [Product Media Contracts](../contracts/product-media-contract.md)
> **Backend Guide:** [Product Media Backend Guide](../backend/product-media-backend-guide.md)

---

## ⚙️ Next.js Configuration (`remotePatterns`)

To guarantee zero code changes when migrating from Phase 1 (database streaming) to Phase 2 (production CDN), `next.config.ts` defines remote patterns for both environments:

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  images: {
    formats: ['image/avif', 'image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
    remotePatterns: [
      // -------------------------------------------------------------
      // Phase 1: Local / Staging NestJS Media Streaming API
      // -------------------------------------------------------------
      {
        protocol: 'http',
        hostname: 'localhost',
        port: '3001',
        pathname: '/api/v1/media/**',
      },
      {
        protocol: 'https',
        hostname: 'api.hobiholidays.com',
        pathname: '/api/v1/media/**',
      },
      // -------------------------------------------------------------
      // Phase 2: Production Cloudflare CDN / AWS S3
      // -------------------------------------------------------------
      {
        protocol: 'https',
        hostname: 'cdn.hobiholidays.com',
        pathname: '/**',
      },
    ],
  },
};

export default nextConfig;
```

---

## 🖼️ Zero-Breaking-Change Media Consumption

Because `media.url` emits a fully-qualified or root-relative URL in both phases, frontend components consume the image URL identically:

```tsx
// components/media/tour-image.tsx
import Image from 'next/image';

interface TourImageProps {
  src: string;
  alt: string;
  aspectRatio?: '16/9' | '16/10' | '4/3' | '1/1';
  priority?: boolean;
}

export function TourImage({
  src,
  alt,
  aspectRatio = '16/10',
  priority = false,
}: TourImageProps) {
  const aspectClass = {
    '16/9': 'aspect-[16/9]',
    '16/10': 'aspect-[16/10]',
    '4/3': 'aspect-[4/3]',
    '1/1': 'aspect-square',
  }[aspectRatio];

  return (
    <div className={`relative w-full overflow-hidden rounded-2xl bg-slate-100 ${aspectClass}`}>
      <Image
        src={src}
        alt={alt}
        fill
        priority={priority}
        sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
        className="object-cover transition-transform duration-300 hover:scale-105"
      />
    </div>
  );
}
```

---

## 📸 Interactive Lightbox & Gallery Carousel

```tsx
// components/media/gallery-carousel.tsx
'use client';

import { useState } from 'react';
import Image from 'next/image';

interface Props {
  productId: string;
}

export function GalleryCarousel({ productId }: Props) {
  const [photos] = useState<string[]>([
    'https://cdn.hobiholidays.com/products/gwe/paris-hero.jpg',
    'https://cdn.hobiholidays.com/products/gwe/swiss-alps.jpg',
    'https://cdn.hobiholidays.com/products/gwe/amsterdam-canal.jpg',
  ]);
  const [activeIdx, setActiveIdx] = useState(0);

  return (
    <div className="space-y-4">
      {/* Featured Large View */}
      <div className="relative aspect-[16/9] w-full rounded-2xl overflow-hidden shadow-sm">
        <Image
          src={photos[activeIdx]}
          alt="Featured Tour Photo"
          fill
          priority
          sizes="(max-width: 1200px) 100vw, 800px"
          className="object-cover"
        />
      </div>

      {/* Thumbnail Navigation Strip */}
      <div className="flex gap-3 overflow-x-auto pb-2">
        {photos.map((url, idx) => (
          <button
            key={url}
            onClick={() => setActiveIdx(idx)}
            className={`relative w-24 aspect-[16/10] rounded-xl overflow-hidden border-2 transition ${
              activeIdx === idx ? 'border-blue-600 ring-2 ring-blue-100' : 'border-transparent opacity-70 hover:opacity-100'
            }`}
          >
            <Image
              src={url}
              alt={`Thumbnail ${idx + 1}`}
              fill
              sizes="96px"
              className="object-cover"
            />
          </button>
        ))}
      </div>
    </div>
  );
}
```
