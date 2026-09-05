# Search & Filter Engine — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the Multi-Criteria Catalog Search & Filter Engine (`GET /api/v1/variants/search`). Covers parameterized dynamic SQL query building, Trigram GIN partial text search (`pg_trgm`), 4-tier geographic joins (`Continent → Sub Continent → Country → POI`), 2-tier Category filtering, Adult pricing evaluations, and in-query window function result counting (`COUNT(*) OVER() AS total_packages`).
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
│   └── search-filter.controller.ts       # GET /variants/search & GET /variants/search/filter-options
├── services/
│   ├── search-filter.service.ts          # Search orchestration, filter-options aggregation & Redis caching
│   └── search-sql-builder.service.ts     # Dynamic parameterized SQL composer
└── dto/
    ├── search-trip.dto.ts                # Incoming search query validation
    └── filter-options-response.dto.ts    # Aggregated filter master data schema
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

    // 1. Category Filters (2-tier taxonomy)
    const category = dto.categorySlug || dto.categoryId;
    if (category) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_categories cat
          WHERE cat.id = p.category_id
            AND (cat.id::text = $${idx} OR cat.slug = $${idx})
        )
      `);
      values.push(category);
      idx++;
    }

    const parentCategory = dto.parentCategorySlug || dto.parentCategoryId;
    if (parentCategory) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_categories pcat
          WHERE pcat.id = p.parent_category_id
            AND (pcat.id::text = $${idx} OR pcat.slug = $${idx})
        )
      `);
      values.push(parentCategory);
      idx++;
    }

    // 2. Geographic Filters (4-tier Area hierarchy: POI -> Country -> Sub-Continent -> Continent)
    const continent = dto.continentSlug || dto.continentId;
    if (continent) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas poi ON poi.id = pl.area_id
          INNER JOIN areas country ON country.id = poi.parent_id
          INNER JOIN areas sub ON sub.id = country.parent_id
          INNER JOIN areas cont ON cont.id = sub.parent_id
          WHERE pl.product_id = p.id
            AND (cont.id::text = $${idx} OR cont.slug = $${idx})
        )
      `);
      values.push(continent);
      idx++;
    }

    const subContinent = dto.subContinentSlug || dto.subContinentId;
    if (subContinent) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas poi ON poi.id = pl.area_id
          INNER JOIN areas country ON country.id = poi.parent_id
          INNER JOIN areas sub ON sub.id = country.parent_id
          WHERE pl.product_id = p.id
            AND (sub.id::text = $${idx} OR sub.slug = $${idx})
        )
      `);
      values.push(subContinent);
      idx++;
    }

    const country = dto.countrySlug || dto.countryId;
    if (country) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas poi ON poi.id = pl.area_id
          INNER JOIN areas country ON country.id = poi.parent_id
          WHERE pl.product_id = p.id
            AND (country.id::text = $${idx} OR country.slug = $${idx})
        )
      `);
      values.push(country);
      idx++;
    }

    const poi = dto.poiSlug || dto.poiId;
    if (poi) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_locations pl
          INNER JOIN areas poi ON poi.id = pl.area_id
          WHERE pl.product_id = p.id
            AND (poi.id::text = $${idx} OR poi.slug = $${idx})
        )
      `);
      values.push(poi);
      idx++;
    }

    // 3. Text Search: Product Name or Variant Title (Trigram Search)
    const searchName = dto.productName;
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
    const totalPack = dto.totalPack || dto.pax;
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

    // 7. Budget Range (minPrice / maxPrice evaluated on ADULT age band)
    if (dto.minPrice) {
      conditions.push(`
        EXISTS (
          SELECT 1 FROM product_trips pt
          INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
          WHERE pt.variant_id = v.id
            AND pt.status = 'ACTIVE'
            AND pt.start_date >= CURRENT_DATE
            AND ptp.age_band = 'ADULT'
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
            AND ptp.age_band = 'ADULT'
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
        cat.id AS category_id,
        cat.name AS category_name,
        cat.slug AS category_slug,
        pcat.id AS parent_category_id,
        pcat.name AS parent_category_name,
        pcat.slug AS parent_category_slug,
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
            AND ptp.age_band = 'ADULT'
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
      LEFT JOIN product_categories cat ON cat.id = p.category_id
      LEFT JOIN product_categories pcat ON pcat.id = p.parent_category_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id
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
      variantId: r.variant_id,
      variantName: r.variant_name,
      variantSlug: r.variant_slug,
      variantType: r.variant_type,
      productId: r.product_id,
      productName: r.product_name,
      productSlug: r.product_slug,
      category: r.category_id ? {
        id: r.category_id,
        name: r.category_name,
        slug: r.category_slug,
      } : null,
      parentCategory: r.parent_category_id ? {
        id: r.parent_category_id,
        name: r.parent_category_name,
        slug: r.parent_category_slug,
      } : null,
      durationDays: r.duration_days,
      durationNights: r.duration_nights,
      startingPrice: r.starting_price ? parseFloat(r.starting_price) : 0,
      currency: 'IDR',
      nextDepartureDate: r.next_departure_date,
      availableDates: r.next_departure_date ? [r.next_departure_date] : [],
      coverImageUrl: r.cover_image_url || 'https://cdn.hobiholidays.com/defaults/cover.jpg',
      destinations: r.destinations || [],
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
        totalPackages,
      },
      data: packages,
    };
  }
}
```

---

## 🎛️ Filter Options Master Data Aggregation

To power catalog sidebar filters and the "Where To?" search widget without executing wasteful multi-table joins on every keystroke, the backend aggregates all active filter dimensions under a single endpoint with multi-tier Redis caching.

### 1. Controller Endpoint Definition

```typescript
// controllers/search-filter.controller.ts
import { Controller, Get, Query, Header } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { SearchFilterService } from '../services/search-filter.service';
import { SearchTripDto } from '../dto/search-trip.dto';
import { FilterOptionsResponseDto } from '../dto/filter-options-response.dto';

@ApiTags('Catalog Search & Filter')
@Controller('api/v1/variants/search')
export class SearchFilterController {
  constructor(private readonly searchFilterService: SearchFilterService) {}

  @Get('filter-options')
  @ApiOperation({
    summary: 'Retrieve dynamic master data filter options for catalog search',
    description: 'Returns only destinations, categories, and departure windows with active tour packages.',
  })
  @ApiResponse({ status: 200, type: FilterOptionsResponseDto })
  @Header('Cache-Control', 'public, s-maxage=300, stale-while-revalidate=600')
  async getFilterOptions() {
    const data = await this.searchFilterService.getFilterOptions();
    return {
      statusCode: 200,
      message: 'Filter options retrieved successfully',
      data,
    };
  }

  @Get()
  @ApiOperation({ summary: 'Multi-criteria catalog search query' })
  @Header('Cache-Control', 'public, s-maxage=60, stale-while-revalidate=300')
  async search(@Query() dto: SearchTripDto) {
    return this.searchFilterService.search(dto);
  }
}
```

---

### 2. DTO Definition

```typescript
// dto/filter-options-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class DestinationPoiOptionDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440003' })
  id: string;

  @ApiProperty({ example: 'Keukenhof Gardens' })
  name: string;

  @ApiProperty({ example: 'keukenhof' })
  slug: string;

  @ApiProperty({ example: 'POI' })
  areaType: 'POI';

  @ApiProperty({ example: 4 })
  activePackagesCount: number;
}

export class DestinationCountryOptionDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440002' })
  id: string;

  @ApiProperty({ example: 'Netherlands' })
  name: string;

  @ApiProperty({ example: 'netherlands' })
  slug: string;

  @ApiProperty({ example: 'COUNTRY' })
  areaType: 'COUNTRY';

  @ApiProperty({ example: 6 })
  activePackagesCount: number;

  @ApiProperty({ type: [DestinationPoiOptionDto] })
  pois: DestinationPoiOptionDto[];
}

export class DestinationContinentOptionDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440001' })
  id: string;

  @ApiProperty({ example: 'Europe' })
  name: string;

  @ApiProperty({ example: 'europe' })
  slug: string;

  @ApiProperty({ example: 'CONTINENT' })
  areaType: 'CONTINENT';

  @ApiProperty({ example: 14 })
  activePackagesCount: number;

  @ApiProperty({ type: [DestinationCountryOptionDto] })
  countries: DestinationCountryOptionDto[];
}

export class CategoryChildOptionDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440011' })
  id: string;

  @ApiProperty({ example: 'Classic Series' })
  name: string;

  @ApiProperty({ example: 'classic-series' })
  slug: string;

  @ApiProperty({ example: 8 })
  activePackagesCount: number;
}

export class CategoryParentOptionDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440010' })
  id: string;

  @ApiProperty({ example: 'Tour Series' })
  name: string;

  @ApiProperty({ example: 'tour-series' })
  slug: string;

  @ApiProperty({ type: [CategoryChildOptionDto] })
  children: CategoryChildOptionDto[];
}

export class PriceRangeOptionDto {
  @ApiProperty({ example: 'IDR' })
  currency: string;

  @ApiProperty({ example: 18500000.0 })
  min: number;

  @ApiProperty({ example: 48000000.0 })
  max: number;
}

export class DepartureMonthOptionDto {
  @ApiProperty({ example: '2026-08' })
  value: string;

  @ApiProperty({ example: 'August 2026' })
  label: string;

  @ApiProperty({ example: 5 })
  activeTripsCount: number;
}

export class VariantTypeOptionDto {
  @ApiProperty({ example: 'STANDARD' })
  key: string;

  @ApiProperty({ example: 'Standard All-Year' })
  label: string;

  @ApiProperty({ example: 6 })
  count: number;
}

export class FilterOptionsResponseDto {
  @ApiProperty({ type: [DestinationContinentOptionDto] })
  destinations: DestinationContinentOptionDto[];

  @ApiProperty({ type: [CategoryParentOptionDto] })
  categories: CategoryParentOptionDto[];

  @ApiProperty({ type: PriceRangeOptionDto })
  priceRange: PriceRangeOptionDto;

  @ApiProperty({ type: [DepartureMonthOptionDto] })
  departureMonths: DepartureMonthOptionDto[];

  @ApiProperty({ type: [VariantTypeOptionDto] })
  variantTypes: VariantTypeOptionDto[];
}
```

---

### 3. Service Implementation with Redis Caching & In-Memory Tree Assembly

```typescript
// services/search-filter.service.ts (Extension for filter-options)
import { Injectable, Inject } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { FilterOptionsResponseDto } from '../dto/filter-options-response.dto';

@Injectable()
export class SearchFilterService {
  constructor(
    private readonly dataSource: DataSource,
    private readonly sqlBuilder: SearchSqlBuilderService,
    @Inject(CACHE_MANAGER) private readonly cacheManager: Cache,
  ) {}

  /**
   * Returns aggregated filter options with 1-hour Redis caching.
   */
  async getFilterOptions(): Promise<FilterOptionsResponseDto> {
    const cacheKey = 'search_filter_options_v1';
    const cached = await this.cacheManager.get<FilterOptionsResponseDto>(cacheKey);
    if (cached) return cached;

    // Parallel execution of all dimension queries
    const [areasRaw, categoriesRaw, priceRangeRaw, departureMonthsRaw, variantTypesRaw] =
      await Promise.all([
        this.fetchActiveAreaTree(),
        this.fetchActiveCategoryTree(),
        this.fetchPriceRange(),
        this.fetchDepartureMonths(),
        this.fetchVariantTypeCounts(),
      ]);

    const result: FilterOptionsResponseDto = {
      destinations: this.assembleDestinationTree(areasRaw),
      categories: this.assembleCategoryTree(categoriesRaw),
      priceRange: priceRangeRaw,
      departureMonths: departureMonthsRaw,
      variantTypes: variantTypesRaw,
    };

    // Cache in Redis for 1 hour (3600 seconds)
    await this.cacheManager.set(cacheKey, result, 3600 * 1000);
    return result;
  }

  // 1. Active Areas Roll-up Query
  private async fetchActiveAreaTree(): Promise<any[]> {
    const sql = `
      WITH active_product_areas AS (
          SELECT 
              pl.area_id,
              COUNT(DISTINCT v.id) AS package_count
          FROM product_locations pl
          INNER JOIN products p 
              ON p.id = pl.product_id 
             AND p.listing_status = 'ACTIVE' 
             AND p.deleted_at IS NULL
          INNER JOIN product_variants v 
              ON v.product_id = p.id 
             AND v.listing_status = 'ACTIVE' 
             AND v.deleted_at IS NULL
          INNER JOIN product_trips t 
              ON t.variant_id = v.id 
             AND t.status = 'ACTIVE' 
             AND t.start_date >= CURRENT_DATE
          GROUP BY pl.area_id
      )
      SELECT 
          a.id, 
          a.parent_id, 
          a.name, 
          a.slug, 
          at.name AS area_type, 
          a.sort_order,
          COALESCE(apa.package_count, 0) AS direct_package_count
      FROM areas a
      INNER JOIN area_types at ON at.id = a.area_type_id
      LEFT JOIN active_product_areas apa ON apa.area_id = a.id
      WHERE a.deleted_at IS NULL
      ORDER BY at.id ASC, a.sort_order ASC, a.name ASC;
    `;
    return this.dataSource.query(sql);
  }

  // 2. Active Categories Query
  private async fetchActiveCategoryTree(): Promise<any[]> {
    const sql = `
      WITH active_cats AS (
          SELECT 
              p.category_id,
              COUNT(DISTINCT v.id) AS package_count
          FROM products p
          INNER JOIN product_variants v 
              ON v.product_id = p.id 
             AND v.listing_status = 'ACTIVE' 
             AND v.deleted_at IS NULL
          INNER JOIN product_trips t 
              ON t.variant_id = v.id 
             AND t.status = 'ACTIVE' 
             AND t.start_date >= CURRENT_DATE
          WHERE p.listing_status = 'ACTIVE' 
            AND p.deleted_at IS NULL
          GROUP BY p.category_id
      )
      SELECT 
          c.id, 
          c.parent_id, 
          c.name, 
          c.slug,
          COALESCE(ac.package_count, 0) AS package_count
      FROM product_categories c
      LEFT JOIN active_cats ac ON ac.category_id = c.id
      ORDER BY c.name ASC;
    `;
    return this.dataSource.query(sql);
  }

  // 3. Price Range Query (Adult)
  private async fetchPriceRange(): Promise<{ currency: string; min: number; max: number }> {
    const sql = `
      SELECT
          COALESCE(MIN(ptp.selling_price), 0.00) AS min_price,
          COALESCE(MAX(ptp.selling_price), 0.00) AS max_price
      FROM product_trip_pricings ptp
      INNER JOIN product_trips pt ON pt.id = ptp.trip_id
      INNER JOIN product_variants pv ON pv.id = pt.variant_id
      WHERE pt.status = 'ACTIVE'
        AND pt.start_date >= CURRENT_DATE
        AND pv.listing_status = 'ACTIVE'
        AND ptp.age_band = 'ADULT';
    `;
    const rows = await this.dataSource.query(sql);
    return {
      currency: 'IDR',
      min: parseFloat(rows[0]?.min_price || '0'),
      max: parseFloat(rows[0]?.max_price || '0'),
    };
  }

  // 4. Departure Months Query
  private async fetchDepartureMonths(): Promise<any[]> {
    const sql = `
      SELECT
          TO_CHAR(pt.start_date, 'YYYY-MM') AS value,
          TO_CHAR(pt.start_date, 'FMMonth YYYY') AS label,
          COUNT(DISTINCT pt.id) AS active_trips_count
      FROM product_trips pt
      INNER JOIN product_variants pv ON pv.id = pt.variant_id
      WHERE pt.status = 'ACTIVE'
        AND pt.start_date >= CURRENT_DATE
        AND pv.listing_status = 'ACTIVE'
      GROUP BY TO_CHAR(pt.start_date, 'YYYY-MM'), TO_CHAR(pt.start_date, 'FMMonth YYYY')
      ORDER BY value ASC;
    `;
    return this.dataSource.query(sql);
  }

  // 5. Variant Type Counts
  private async fetchVariantTypeCounts(): Promise<any[]> {
    const sql = `
      SELECT 
          v.variant_type AS key,
          COUNT(DISTINCT v.id) AS count
      FROM product_variants v
      INNER JOIN product_trips t ON t.variant_id = v.id
      WHERE v.listing_status = 'ACTIVE'
        AND v.deleted_at IS NULL
        AND t.status = 'ACTIVE'
        AND t.start_date >= CURRENT_DATE
      GROUP BY v.variant_type;
    `;
    const rows = await this.dataSource.query(sql);
    const labels: Record<string, string> = {
      STANDARD: 'Standard All-Year',
      SEASONAL: 'Seasonal Edition',
      THEMED: 'Themed Edition',
      PROMOTIONAL: 'Flash Sale / Promo',
    };
    return rows.map((r: any) => ({
      key: r.key,
      label: labels[r.key] || r.key,
      count: parseInt(r.count, 10),
    }));
  }

  // In-Memory Destination Hierarchy Assembler (Pruning zero-package nodes)
  private assembleDestinationTree(rawAreas: any[]): any[] {
    const continents: any[] = [];
    const countryMap = new Map<string, any>();
    const poiList: any[] = [];

    rawAreas.forEach((area) => {
      if (area.area_type === 'CONTINENT') {
        continents.push({
          id: area.id,
          name: area.name,
          slug: area.slug,
          areaType: 'CONTINENT',
          activePackagesCount: parseInt(area.direct_package_count, 10),
          countries: [],
        });
      } else if (area.area_type === 'COUNTRY') {
        const country = {
          id: area.id,
          parent_id: area.parent_id,
          name: area.name,
          slug: area.slug,
          areaType: 'COUNTRY',
          activePackagesCount: parseInt(area.direct_package_count, 10),
          pois: [],
        };
        countryMap.set(area.id, country);
      } else if (area.area_type === 'POI') {
        poiList.push({
          id: area.id,
          parent_id: area.parent_id,
          name: area.name,
          slug: area.slug,
          areaType: 'POI',
          activePackagesCount: parseInt(area.direct_package_count, 10),
        });
      }
    });

    // Attach POIs to Countries and roll up count
    poiList.forEach((poi) => {
      const country = countryMap.get(poi.parent_id);
      if (country) {
        country.activePackagesCount += poi.activePackagesCount;
        if (poi.activePackagesCount > 0) {
          country.pois.push(poi);
        }
      }
    });

    // Attach Countries to Continents and roll up count
    continents.forEach((continent) => {
      countryMap.forEach((country) => {
        // Direct child or linked via sub-continent
        continent.countries.push(country);
        continent.activePackagesCount += country.activePackagesCount;
      });
    });

    // Filter to retain only continents and countries with active packages > 0
    return continents
      .filter((c) => c.activePackagesCount > 0)
      .map((c) => ({
        ...c,
        countries: c.countries.filter((co: any) => co.activePackagesCount > 0),
      }));
  }

  // In-Memory Category Tree Assembler
  private assembleCategoryTree(rawCats: any[]): any[] {
    const parents: any[] = [];
    const childrenMap = new Map<string, any[]>();

    rawCats.forEach((c) => {
      if (!c.parent_id) {
        parents.push({
          id: c.id,
          name: c.name,
          slug: c.slug,
          children: [],
        });
      } else {
        const list = childrenMap.get(c.parent_id) || [];
        if (parseInt(c.package_count, 10) > 0) {
          list.push({
            id: c.id,
            name: c.name,
            slug: c.slug,
            activePackagesCount: parseInt(c.package_count, 10),
          });
        }
        childrenMap.set(c.parent_id, list);
      }
    });

    return parents
      .map((p) => ({
        ...p,
        children: childrenMap.get(p.id) || [],
      }))
      .filter((p) => p.children.length > 0);
  }
}
```

---

### 4. Cache Invalidation Triggers

The `'search_filter_options_v1'` Redis cache key must be proactively invalidated whenever catalog availability changes:

```typescript
// Example inside ProductService / TripService
@Injectable()
export class ProductMutationListener {
  constructor(@Inject(CACHE_MANAGER) private readonly cacheManager: Cache) {}

  async onCatalogMutated() {
    await this.cacheManager.del('search_filter_options_v1');
  }
}
```

**Events triggering invalidation:**
1. Master product `listing_status` updated or product soft-deleted.
2. Variant `listing_status` toggled (e.g. from `DRAFT` to `ACTIVE`).
3. Trip scheduled, quota updated to full (`status = 'FULL'`), or trip cancelled (`status = 'CANCELLED'`).

