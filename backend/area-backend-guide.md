# Area Domain — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the Area Domain subsystem, managing the **3-tier Geographic Hierarchy** (`Continent → Country → City`). Covers recursive Common Table Expressions (CTE) for tree traversal, PostGIS spatial boundary queries, autocomplete discovery, and in-memory reference caching.
>
> **Related Design Document:** [Area Domain Technical Design](../technical/area-technical-design.md)
> **API Contract:** [Area Contracts](../contracts/area-contract.md)
> **Frontend Guide:** [Area Frontend Guide](../frontend/area-frontend-guide.md)

---

## 🏗️ Module Overview

The Area domain lives under `src/modules/area/`:

```
src/modules/area/
├── area.module.ts
├── controllers/
│   ├── area.controller.ts             # Public autocomplete, tree, and listings
│   └── area-admin.controller.ts       # Administrative CRUD operations
├── services/
│   ├── area.service.ts                # Hierarchical tree & cache resolution
│   └── area-spatial.service.ts        # PostGIS geometric boundary lookups
└── dto/
    ├── search-area-autocomplete.dto.ts
    └── create-area.dto.ts
```

---

## 🌳 Hierarchical Tree Traversal via Recursive CTE

The geographic model uses an adjacency list (`parent_id`) strictly capped at 3 tiers (`CONTINENT → COUNTRY → CITY`). The backend resolves ancestor/descendant paths via a recursive PostgreSQL Common Table Expression (CTE):

```typescript
// services/area.service.ts
import { Injectable, Inject } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class AreaService {
  constructor(
    private readonly dataSource: DataSource,
    @Inject(CACHE_MANAGER) private readonly cacheManager: Cache,
  ) {}

  /**
   * Returns full 3-tier hierarchical tree from cache or database.
   * Static reference data is cached for 24 hours.
   */
  async getHierarchyTree() {
    const cacheKey = 'area_hierarchy_tree_v1';
    const cached = await this.cacheManager.get(cacheKey);
    if (cached) return cached;

    // Fetch all 3 tiers in a single indexed query
    const areas = await this.dataSource.query(`
      SELECT id, parent_id, name, slug, area_type, code, sort_order
      FROM areas
      WHERE status = 'ACTIVE'
      ORDER BY area_type ASC, sort_order ASC, name ASC;
    `);

    // In-memory nesting assembly: O(N) single-pass dictionary assembly
    const continents: any[] = [];
    const countryMap = new Map<string, any>();

    areas.forEach((area: any) => {
      if (area.area_type === 'CONTINENT') {
        continents.push({ ...area, countries: [] });
      } else if (area.area_type === 'COUNTRY') {
        area.cities = [];
        countryMap.set(area.id, area);
      }
    });

    // Attach countries to continents
    countryMap.forEach((country) => {
      const continent = continents.find((c) => c.id === country.parent_id);
      if (continent) {
        continent.countries.push(country);
      }
    });

    // Attach cities to countries
    areas.forEach((area: any) => {
      if (area.area_type === 'CITY') {
        const country = countryMap.get(area.parent_id);
        if (country) {
          country.cities.push(area);
        }
      }
    });

    // Cache the assembled tree
    await this.cacheManager.set(cacheKey, continents, 86400 * 1000); // 24 hours

    return continents;
  }

  /**
   * Search widget autocomplete endpoint (powers the "Where To?" search dropdown)
   */
  async autocomplete(query: string, limit: number = 10) {
    const trimmed = query.trim();
    if (!trimmed || trimmed.length < 2) return [];

    const sql = `
      SELECT
        a.id,
        a.name,
        a.slug,
        a.area_type,
        p.name AS parent_name,
        p.slug AS parent_slug,
        gp.name AS grandparent_name,
        -- Package count active in this area
        (
          SELECT COUNT(DISTINCT pl.product_id)
          FROM product_locations pl
          INNER JOIN products prod ON prod.id = pl.product_id
          WHERE pl.area_id = a.id AND prod.listing_status = 'ACTIVE'
        ) AS active_packages_count
      FROM areas a
      LEFT JOIN areas p ON p.id = a.parent_id
      LEFT JOIN areas gp ON gp.id = p.parent_id
      WHERE a.status = 'ACTIVE'
        AND a.name ILIKE $1
      ORDER BY
        (CASE WHEN a.name ILIKE $2 THEN 1 ELSE 2 END) ASC,
        active_packages_count DESC
      LIMIT $3;
    `;

    return this.dataSource.query(sql, [`%${trimmed}%`, `${trimmed}%`, limit]);
  }
}
```

---

## 🗺️ PostGIS Spatial Boundary Execution

When calculating whether a tour coordinates stop falls inside an administrative territory:

```typescript
// services/area-spatial.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class AreaSpatialService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Checks if a GPS coordinate (latitude, longitude) falls inside
   * an area's administrative boundary polygon.
   */
  async findAreaByCoordinates(lat: number, lng: number) {
    const sql = `
      SELECT id, name, slug, area_type
      FROM areas
      WHERE boundary IS NOT NULL
        AND ST_Contains(boundary, ST_SetSRID(ST_MakePoint($1, $2), 4326))
      ORDER BY
        (CASE area_type
          WHEN 'CITY' THEN 1
          WHEN 'COUNTRY' THEN 2
          WHEN 'CONTINENT' THEN 3
          ELSE 4 END) ASC
      LIMIT 1;
    `;

    const rows = await this.dataSource.query(sql, [lng, lat]);
    return rows[0] || null;
  }
}
```
