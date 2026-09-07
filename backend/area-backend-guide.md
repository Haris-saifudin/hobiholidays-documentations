# Area Domain — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the Area Domain subsystem, managing the **4-tier Geographic Hierarchy** (`Continent → Sub Continent → Country → POI`). Covers hierarchical tree assembly, standard WGS-84 coordinate mapping, autocomplete discovery, and in-memory reference caching.
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
│   ├── area.controller.ts             # Public autocomplete, 4-tier tree, and listings
│   └── area-admin.controller.ts       # Administrative CRUD operations
├── services/
│   └── area.service.ts                # Hierarchical tree & cache resolution
└── dto/
    ├── search-area-autocomplete.dto.ts
    └── create-area.dto.ts
```

---

## 🌳 Hierarchical Tree Traversal & In-Memory Assembly

The geographic model uses an adjacency list (`parent_id`) structured across 4 tiers (`CONTINENT → SUB_CONTINENT → COUNTRY → POI`). The backend resolves the tree in a single-pass in-memory dictionary assembly with 24-hour Redis caching:

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
   * Returns full 4-tier hierarchical tree from cache or database.
   * Static reference data is cached for 24 hours.
   */
  async getHierarchyTree() {
    const cacheKey = 'area_hierarchy_tree_v2';
    const cached = await this.cacheManager.get(cacheKey);
    if (cached) return cached;

    // Fetch all 4 tiers in a single indexed query
    const areas = await this.dataSource.query(`
      SELECT a.id, a.parent_id, a.name, a.slug, at.name AS area_type, a.code, a.lat, a.lng, a.sort_order
      FROM areas a
      INNER JOIN area_types at ON at.id = a.area_type_id
      WHERE a.deleted_at IS NULL
      ORDER BY a.area_type_id ASC, a.sort_order ASC, a.name ASC;
    `);

    // In-memory nesting assembly: O(N) single-pass dictionary assembly
    const continents: any[] = [];
    const subContinentMap = new Map<string, any>();
    const countryMap = new Map<string, any>();

    areas.forEach((area: any) => {
      if (area.area_type === 'CONTINENT') {
        continents.push({ ...area, subContinents: [] });
      } else if (area.area_type === 'SUB_CONTINENT') {
        area.countries = [];
        subContinentMap.set(area.id, area);
      } else if (area.area_type === 'COUNTRY') {
        area.pois = [];
        countryMap.set(area.id, area);
      }
    });

    // Attach sub-continents to continents
    subContinentMap.forEach((subContinent) => {
      const continent = continents.find((c) => c.id === subContinent.parent_id);
      if (continent) {
        continent.subContinents.push(subContinent);
      }
    });

    // Attach countries to sub-continents
    countryMap.forEach((country) => {
      const subContinent = subContinentMap.get(country.parent_id);
      if (subContinent) {
        subContinent.countries.push(country);
      }
    });

    // Attach POIs to countries
    areas.forEach((area: any) => {
      if (area.area_type === 'POI') {
        const country = countryMap.get(area.parent_id);
        if (country) {
          country.pois.push(area);
        }
      }
    });

    // Cache the assembled tree
    await this.cacheManager.set(cacheKey, continents, 86400 * 1000); // 24 hours

    return continents;
  }

  /**
   * Search widget autocomplete endpoint (powers the "Where To?" search dropdown)
   * Matches across Continents, Sub Continents, Countries, and POIs.
   */
  async autocomplete(query: string, limit: number = 10) {
    const trimmed = query.trim();
    if (!trimmed || trimmed.length < 2) return [];

    const sql = `
      SELECT
        a.id,
        a.name,
        a.slug,
        at.name AS area_type,
        a.lat,
        a.lng,
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
      INNER JOIN area_types at ON at.id = a.area_type_id
      LEFT JOIN areas p ON p.id = a.parent_id
      LEFT JOIN areas gp ON gp.id = p.parent_id
      WHERE a.deleted_at IS NULL
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
