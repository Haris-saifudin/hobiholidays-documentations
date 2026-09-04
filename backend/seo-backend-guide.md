# SEO & Structured Data — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the SEO, Open Graph metadata, and Schema.org Structured Data subsystem. Covers polymorphic metadata persistence, dual-layer fallback resolution (Layer A custom DB vs Layer B formulaic fallback), and server-side Schema.org JSON-LD graph generation.
>
> **Related Design Document:** [SEO Technical Design](../technical/seo-technical-design.md)
> **API Contract:** [SEO Contracts](../contracts/seo-contract.md)
> **Frontend Guide:** [SEO Frontend Guide](../frontend/seo-frontend-guide.md)

---

## 🏗️ Module Architecture

```
src/modules/seo/
├── seo.module.ts
├── controllers/
│   └── seo.controller.ts             # Polymorphic GET/PUT/DELETE /api/v1/seo/:targetType/:targetId
├── services/
│   ├── seo.service.ts                # Dual-layer resolution & metadata CRUD
│   └── jsonld-builder.service.ts     # Schema.org TouristTrip & Offer JSON-LD builder
└── dto/
    └── upsert-seo-metadata.dto.ts
```

---

## ⚡ Dual-Layer Fallback Resolution Engine

When a client or frontend requests SEO tags for a Product, Variant, or Area, the service checks for explicit custom overrides in `seo_metadata`. If missing or empty, it dynamically generates keyword-rich metadata using standardized fallback formulas:

```typescript
// services/seo.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { UpsertSeoMetadataDto } from '../dto/upsert-seo-metadata.dto';

@Injectable()
export class SeoService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Resolves SEO metadata: returns custom DB record if present,
   * otherwise executes the dynamic formulaic fallback algorithm.
   */
  async resolveSeo(targetType: 'PRODUCT' | 'VARIANT' | 'AREA', targetId: string) {
    // 1. Check Layer A: Explicit custom override in seo_metadata
    const customRows = await this.dataSource.query(`
      SELECT * FROM seo_metadata
      WHERE target_type = $1 AND target_id = $2
      LIMIT 1;
    `, [targetType, targetId]);

    if (customRows.length > 0 && customRows[0].meta_title) {
      const row = customRows[0];
      return {
        targetType,
        targetId,
        isCustom: true,
        metaTitle: row.meta_title,
        metaDescription: row.meta_description,
        canonicalUrl: row.canonical_url,
        ogTitle: row.og_title || row.meta_title,
        ogDescription: row.og_description || row.meta_description,
        ogImageUrl: row.og_image_url,
        noIndex: row.no_index,
        noFollow: row.no_follow,
        structuredData: row.structured_data,
      };
    }

    // 2. Execute Layer B: Programmatic dynamic fallback
    return this.generateDynamicFallback(targetType, targetId);
  }

  private async generateDynamicFallback(targetType: string, targetId: string) {
    if (targetType === 'VARIANT') {
      const rows = await this.dataSource.query(`
        SELECT
          v.name AS variant_name, v.slug AS variant_slug,
          COALESCE(v.duration_days, pj.duration_days) AS duration_days,
          COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
          p.name AS product_name, p.slug AS product_slug,
          (
            SELECT MIN(ptp.selling_price)
            FROM product_trips pt
            INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
            WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
          ) AS starting_price,
          (
            SELECT m.url FROM product_media_usages pmu
            INNER JOIN product_media m ON m.id = pmu.media_id
            WHERE (pmu.target_type = 'VARIANT' AND pmu.target_id = v.id AND pmu.usage_context = 'COVER')
               OR (pmu.target_type = 'PRODUCT' AND pmu.target_id = p.id AND pmu.usage_context = 'COVER')
            LIMIT 1
          ) AS cover_url
        FROM product_variants v
        INNER JOIN products p ON p.id = v.product_id
        LEFT JOIN product_journeys pj ON pj.product_id = p.id AND pj.nationality_scope = 'ALL'
        WHERE v.id = $1
        LIMIT 1;
      `, [targetId]);

      if (!rows.length) throw new NotFoundException(`Variant '${targetId}' not found`);
      const v = rows[0];

      const priceFmt = v.starting_price
        ? `Mulai IDR ${new Intl.NumberFormat('id-ID').format(v.starting_price)}`
        : 'Hubungi Kami';

      const metaTitle = `${v.variant_name} (${v.duration_days}D/${v.duration_nights}N) | Hobiholidays`;
      const metaDescription = `Ikuti tour ${v.variant_name} (${v.duration_days} Hari). ${priceFmt}. Paket tour terpercaya bersama Hobiholidays.`;
      const canonicalUrl = `https://www.hobiholidays.com/tours/${v.product_slug}/${v.variant_slug}`;

      return {
        targetType,
        targetId,
        isCustom: false,
        metaTitle,
        metaDescription,
        canonicalUrl,
        ogTitle: metaTitle,
        ogDescription: metaDescription,
        ogImageUrl: v.cover_url || 'https://cdn.hobiholidays.com/defaults/og-tour-default.jpg',
        noIndex: false,
        noFollow: false,
        structuredData: null,
      };
    }

    if (targetType === 'AREA') {
      const rows = await this.dataSource.query(`
        SELECT name, slug, area_type FROM areas WHERE id = $1 LIMIT 1
      `, [targetId]);

      if (!rows.length) throw new NotFoundException(`Area '${targetId}' not found`);
      const a = rows[0];

      const metaTitle = `Paket Tour & Wisata ke ${a.name} Terbaik | Hobiholidays`;
      const metaDescription = `Temukan pilihan paket tour terlengkap ke ${a.name}. Jadwal keberangkatan pasti dan harga terbaik bersama Hobiholidays.`;
      const canonicalUrl = `https://www.hobiholidays.com/destinations/${a.slug}`;

      return {
        targetType,
        targetId,
        isCustom: false,
        metaTitle,
        metaDescription,
        canonicalUrl,
        ogTitle: metaTitle,
        ogDescription: metaDescription,
        ogImageUrl: 'https://cdn.hobiholidays.com/defaults/og-area-default.jpg',
        noIndex: false,
        noFollow: false,
        structuredData: null,
      };
    }

    throw new NotFoundException(`Unsupported target type: ${targetType}`);
  }

  /**
   * Upserts custom SEO override
   */
  async upsert(targetType: string, targetId: string, dto: UpsertSeoMetadataDto) {
    const sql = `
      INSERT INTO seo_metadata (
        target_type, target_id, meta_title, meta_description, canonical_url,
        og_title, og_description, og_image_url, no_index, no_follow, structured_data,
        updated_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, NOW())
      ON CONFLICT (target_type, target_id) DO UPDATE SET
        meta_title = EXCLUDED.meta_title,
        meta_description = EXCLUDED.meta_description,
        canonical_url = EXCLUDED.canonical_url,
        og_title = EXCLUDED.og_title,
        og_description = EXCLUDED.og_description,
        og_image_url = EXCLUDED.og_image_url,
        no_index = EXCLUDED.no_index,
        no_follow = EXCLUDED.no_follow,
        structuredData = EXCLUDED.structured_data,
        updated_at = NOW()
      RETURNING *;
    `;

    const values = [
      targetType,
      targetId,
      dto.metaTitle || null,
      dto.metaDescription || null,
      dto.canonicalUrl || null,
      dto.ogTitle || dto.metaTitle || null,
      dto.ogDescription || dto.metaDescription || null,
      dto.ogImageUrl || null,
      dto.noIndex ?? false,
      dto.noFollow ?? false,
      dto.structuredData ? JSON.stringify(dto.structuredData) : null,
    ];

    const result = await this.dataSource.query(sql, values);
    return result[0];
  }
}
```

---

## 🏷️ Schema.org Structured Data Generator

```typescript
// services/jsonld-builder.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()
export class JsonLdBuilderService {
  buildTouristTripSchema(variant: any, trips: any[]) {
    const offers = trips.map((t) => ({
      '@type': 'Offer',
      price: t.selling_price,
      priceCurrency: 'IDR',
      availability: t.available_seats > 0
        ? 'https://schema.org/InStock'
        : 'https://schema.org/SoldOut',
      validFrom: new Date().toISOString().split('T')[0],
      url: `https://www.hobiholidays.com/tours/${variant.product_slug}/${variant.slug}`,
    }));

    return {
      '@context': 'https://schema.org',
      '@type': 'TouristTrip',
      name: variant.name,
      description: variant.description,
      touristType: ['Family', 'Group', 'Leisure'],
      itinerary: {
        '@type': 'ItemList',
        numberOfItems: variant.duration_days,
      },
      offers,
    };
  }
}
```
