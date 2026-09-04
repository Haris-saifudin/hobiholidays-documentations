# Search & Filter Engine — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the Multi-Criteria Catalog Search & Filter Engine (`GET /api/v1/variants/search`). Covers parameterized dynamic SQL query building, Trigram GIN partial text search (`pg_trgm`), multi-tier geographic joins, and in-query window function result counting (`COUNT(*) OVER() AS total_packages`).
>
> **Related Design Document:** [Search & Filter Architecture](../technical/product-search-filter-technical-design.md)
> **API Contract:** [Search & Filter Contracts](../contracts/product-search-filter-contract.md)
> **Frontend Guide:** [Search & Filter Frontend Guide](../frontend/product-search-filter-frontend-guide.md)

---

## 🏗️ Module Architecture

```
src/modules/search/
├── search-filter.module.ts
├── controllers/
│   └── search-filter.controller.ts       # GET /api/v1/variants/search
├── services/
│   ├── search-filter.service.ts          # Query orchestration & response formatting
│   └── search-sql-builder.service.ts     # Dynamic parameterized SQL composer
└── dto/
    └── search-trip.dto.ts
```

---

## ⚙️ Dynamic Parameterized SQL Query Composer

The search engine executes a single unified SQL query with dynamic filter parameters to maximize performance and guarantee zero SQL injection vulnerabilities:

```typescript
// services/search-sql-builder.service.ts
import { Injectable } from '@nestjs/common';
import { SearchTripDto } from '../dto/search-trip.dto';

@Injectable()
export class SearchSqlBuilderService {
  buildSearchQuery(dto: SearchTripDto): { sql: string; values: any[] } {
    const conditions: string[] = [
      `v.listing_status = 'ACTIVE'`,
      `p.listing_status = 'ACTIVE'`,
      `p.deleted_at IS NULL`,
    ];
    const values: any[] = [];
    let idx = 1;

    // 1. Geographic Filter: Continent (by UUID or slug)
    const continent = dto.continent || (dto as any).continentSlug || (dto as any).continentId;
    if (continent) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas city ON city.id = pl.area_id
          INNER JOIN areas country ON country.id = city.parent_id
          INNER JOIN areas cont ON cont.id = country.parent_id
          WHERE pl.product_id = p.id
            AND (cont.id::text = $${idx} OR cont.slug = $${idx})
        )
      `);
      values.push(continent);
      idx++;
    }

    // 2. Geographic Filter: Country (by UUID or slug)
    const country = dto.country || (dto as any).countrySlug || (dto as any).countryId;
    if (country) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas city ON city.id = pl.area_id
          INNER JOIN areas country ON country.id = city.parent_id
          WHERE pl.product_id = p.id
            AND (country.id::text = $${idx} OR country.slug = $${idx})
        )
      `);
      values.push(country);
      idx++;
    }

    // 3. Text Search: Product Name or Variant Title (Trigram Search)
    const searchName = dto.name || (dto as any).productName;
    if (searchName) {
      conditions.push(`(p.name ILIKE $${idx} OR v.name ILIKE $${idx})`);
      values.push(`%${searchName.trim()}%`);
      idx++;
    }

    // 4. Variant Type Classification
    if (dto.variantType) {
      conditions.push(`v.variant_type = $${idx}`);
      values.push(dto.variantType);
      idx++;
    }

    // 5. Quota (Total Pack / Pax Capacity)
    const totalPack = dto.totalPack || (dto as any).pax;
    if (totalPack) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_trips pt
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
            AND pt.min_quota <= $${idx}
            AND pt.max_quota >= $${idx}
        )
      `);
      values.push(totalPack);
      idx++;
    }

    // 6. Departure Month (YYYY-MM)
    if (dto.departureMonth) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_trips pt
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND TO_CHAR(pt.start_date, 'YYYY-MM') = $${idx}
        )
      `);
      values.push(dto.departureMonth);
      idx++;
    }

    // 7. Budget Range (minPrice / maxPrice)
    if (dto.minPrice) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_trips pt
          INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
            AND ptp.selling_price >= $${idx}
        )
      `);
      values.push(dto.minPrice);
      idx++;
    }

    if (dto.maxPrice) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_trips pt
          INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
            AND ptp.selling_price <= $${idx}
        )
      `);
      values.push(dto.maxPrice);
      idx++;
    }

    const page = dto.page || 1;
    const limit = dto.limit || 10;
    const offset = (page - 1) * limit;

    values.push(limit);
    const limitIdx = idx++;
    values.push(offset);
    const offsetIdx = idx++;

    const sql = `
      SELECT
        v.id AS variant_id,
        v.code AS variant_code,
        v.name AS variant_name,
        v.slug AS variant_slug,
        v.variant_type,
        COALESCE(v.duration_days, pj.duration_days) AS duration_days,
        COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
        p.id AS product_id,
        p.name AS product_name,
        p.slug AS product_slug,
        p.itinerary_pdf_url,
        (
          SELECT m.url FROM product_media_usages pmu
          INNER JOIN product_media m ON m.id = pmu.media_id
          WHERE (pmu.target_type = 'VARIANT' AND pmu.target_id = v.id AND pmu.usage_context = 'COVER')
             OR (pmu.target_type = 'PRODUCT' AND pmu.target_id = p.id AND pmu.usage_context = 'COVER')
          ORDER BY (CASE WHEN pmu.target_type = 'VARIANT' THEN 1 ELSE 2 END) ASC
          LIMIT 1
        ) AS cover_image_url,
        (
          SELECT MIN(ptp.selling_price)
          FROM product_trips pt
          INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
        ) AS starting_price,
        (
          SELECT MIN(pt.start_date)
          FROM product_trips pt
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
        ) AS next_departure_date,
        COUNT(*) OVER() AS total_packages
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id AND pj.nationality_scope = 'ALL'
      WHERE ${conditions.join(' AND ')}
      ORDER BY starting_price ASC NULLS LAST, v.created_at DESC
      LIMIT $${limitIdx} OFFSET $${offsetIdx};
    `;

    return { sql, values };
  }
}
```

---

## 🎮 Service Execution & Response Envelope

```typescript
// services/search-filter.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { SearchSqlBuilderService } from './search-sql-builder.service';
import { SearchTripDto } from '../dto/search-trip.dto';

@Injectable()
export class SearchFilterService {
  constructor(
    private readonly dataSource: DataSource,
    private readonly sqlBuilder: SearchSqlBuilderService,
  ) {}

  async search(dto: SearchTripDto) {
    const { sql, values } = this.sqlBuilder.buildSearchQuery(dto);
    const rows = await this.dataSource.query(sql, values);

    const totalPackages = rows.length > 0 ? parseInt(rows[0].total_packages, 10) : 0;
    const page = dto.page || 1;
    const limit = dto.limit || 10;
    const totalPages = Math.ceil(totalPackages / limit);

    const packages = rows.map((r) => ({
      id: r.variant_id,
      variantId: r.variant_id,
      code: r.variant_code,
      name: r.variant_name,
      variantName: r.variant_name,
      slug: r.variant_slug,
      variantSlug: r.variant_slug,
      variantType: r.variant_type,
      durationDays: r.duration_days,
      durationNights: r.duration_nights,
      startingPrice: r.starting_price ? parseFloat(r.starting_price) : 0,
      currency: 'IDR',
      nextDepartureDate: r.next_departure_date,
      availableDates: r.available_dates || (r.next_departure_date ? [r.next_departure_date] : []),
      coverImageUrl: r.cover_image_url || 'https://cdn.hobiholidays.com/defaults/cover.jpg',
      destinations: r.destinations || [],
      productId: r.product_id,
      productName: r.product_name,
      productSlug: r.product_slug,
      product: {
        id: r.product_id,
        name: r.product_name,
        slug: r.product_slug,
        itineraryPdfUrl: r.itinerary_pdf_url,
      },
    }));

    return {
      statusCode: 200,
      message: `Search completed: ${totalPackages} tour packages found`,
      meta: {
        totalItems: totalPackages,
        itemCount: packages.length,
        itemsPerPage: limit,
        totalPages,
        currentPage: page,
        totalPackages, // Preserved for backwards compatibility
      },
      data: packages,
    };
  }
}
```
