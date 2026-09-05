# Area Domain API Contracts

> **Overview**
> API contract specifications for the Area/Geography domain, managing the **4-tier geographic hierarchy** (`Continent → Sub Continent → Country → POI`), search widget autocomplete, and destination landing page metadata.
>
> **Related Design Document:** [Area Domain Technical Design](../technical/area-technical-design.md)  
> **Backend Guide:** [Area Backend Guide](../backend/area-backend-guide.md)  
> **Frontend Guide:** [Area Frontend Guide](../frontend/area-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Area Types** | `GET` | `/api/v1/areas/types` | List area classifications (`CONTINENT`, `SUB_CONTINENT`, `COUNTRY`, `POI`) |
| **Hierarchy Navigation** | `GET` | `/api/v1/areas/continents` | List all continents (root nodes) |
| | `GET` | `/api/v1/areas/continents/:continentId/sub-continents` | List sub-continents within a continent |
| | `GET` | `/api/v1/areas/sub-continents/:subContinentId/countries` | List countries within a sub-continent |
| | `GET` | `/api/v1/areas/countries/:countryId/pois` | List points of interest (POIs) within a country |
| | `GET` | `/api/v1/areas/tree` | Fetch complete global 4-tier hierarchy tree (Continents → Sub Continents → Countries → POIs) |
| | `GET` | `/api/v1/areas/:id/tree` | Fetch complete 4-tier ancestor/descendant tree rooted at node |
| **Search Widget** | `GET` | `/api/v1/areas/autocomplete` | Fast autocomplete matching continent, sub-continent, country, or POI |
| **Destination Landing** | `GET` | `/api/v1/areas/:slug` | Destination hub page with tours count and embedded SEO |
| **Admin CRUD** | `POST` | `/api/v1/areas` | Create geographic area node |
| | `PUT` | `/api/v1/areas/:id` | Update geographic area |
| | `DELETE`| `/api/v1/areas/:id` | Soft delete area node |

---

## 1. Area Types (`GET /api/v1/areas/types`)

Returns the full list of area classifications in the 4-tier taxonomy. This is a small, static reference list — no pagination needed.

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area types retrieved successfully",
  "data": [
    {
      "id": 1,
      "name": "CONTINENT",
      "description": "Global continental landmasses and geographic macro-regions (Tier 1 root level)"
    },
    {
      "id": 2,
      "name": "SUB_CONTINENT",
      "description": "Geographical sub-regions within a continent (Tier 2)"
    },
    {
      "id": 3,
      "name": "COUNTRY",
      "description": "Sovereign states and independent nations (Tier 3)"
    },
    {
      "id": 4,
      "name": "POI",
      "description": "Points of Interest, landmark attractions, and destination highlights (Tier 4 leaf level)"
    }
  ]
}
```

---

## 2. Search Widget Autocomplete (`GET /api/v1/areas/autocomplete`)

Powers the **"Where To?"** search widget dropdown on the homepage and All Tours navigation.

### Request Query Parameters
- `q`: Search keyword (min 2 characters, e.g. `jap`, `tokyo`, or `eiffel`)
- `limit`: Max results (default 8)

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Autocomplete results retrieved",
  "data": [
    {
      "areaId": "550e8400-e29b-41d4-a716-446655440003",
      "name": "Japan",
      "slug": "japan",
      "areaType": "COUNTRY",
      "parentName": "East Asia",
      "displayName": "Japan (East Asia, Asia)",
      "activeToursCount": 12
    },
    {
      "areaId": "550e8400-e29b-41d4-a716-446655440004",
      "name": "Mount Fuji",
      "slug": "mount-fuji",
      "areaType": "POI",
      "parentName": "Japan",
      "displayName": "Mount Fuji, Japan",
      "activeToursCount": 8
    },
    {
      "areaId": "550e8400-e29b-41d4-a716-446655440006",
      "name": "Western Europe",
      "slug": "western-europe",
      "areaType": "SUB_CONTINENT",
      "parentName": "Europe",
      "displayName": "Western Europe (Europe)",
      "activeToursCount": 35
    }
  ]
}
```

---

## 3. Hierarchy Navigation Endpoints

### 3.1 List Continents (`GET /api/v1/areas/continents`)

Returns paginated list of root-level continental nodes (Tier 1).

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `10`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Continents retrieved successfully",
  "meta": {
    "totalItems": 6,
    "itemCount": 2,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "code": "ASIA",
      "name": "Asia",
      "slug": "asia",
      "listingStatus": "ACTIVE",
      "subContinentsCount": 5,
      "countriesCount": 48,
      "toursCount": 120,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440005",
      "code": "EUROPE",
      "name": "Europe",
      "slug": "europe",
      "listingStatus": "ACTIVE",
      "subContinentsCount": 4,
      "countriesCount": 44,
      "toursCount": 85,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.2 List Sub-Continents in Continent (`GET /api/v1/areas/continents/:continentId/sub-continents`)

Returns paginated list of sub-continents (Tier 2) under a specific continent.

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `20`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Sub-continents retrieved successfully",
  "meta": {
    "totalItems": 4,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440006",
      "code": "WEST-EUR",
      "name": "Western Europe",
      "slug": "western-europe",
      "listingStatus": "ACTIVE",
      "countriesCount": 9,
      "toursCount": 35,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440007",
      "code": "SOUTH-EUR",
      "name": "Southern Europe",
      "slug": "southern-europe",
      "listingStatus": "ACTIVE",
      "countriesCount": 15,
      "toursCount": 28,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.3 List Countries in Sub-Continent (`GET /api/v1/areas/sub-continents/:subContinentId/countries`)

Returns paginated list of sovereign countries (Tier 3) under a specific sub-continent.

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `20`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Countries retrieved successfully",
  "meta": {
    "totalItems": 9,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440008",
      "code": "FR",
      "name": "France",
      "slug": "france",
      "isoCode": "FR",
      "listingStatus": "ACTIVE",
      "poisCount": 14,
      "toursCount": 22,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440009",
      "code": "NL",
      "name": "Netherlands",
      "slug": "netherlands",
      "isoCode": "NL",
      "listingStatus": "ACTIVE",
      "poisCount": 8,
      "toursCount": 18,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.4 List POIs in Country (`GET /api/v1/areas/countries/:countryId/pois`)

Returns paginated list of points of interest (Tier 4 leaf nodes) under a specific country.

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `20`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "POIs retrieved successfully",
  "meta": {
    "totalItems": 14,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "code": "FR-PAR-EIFFEL",
      "name": "Eiffel Tower",
      "slug": "eiffel-tower",
      "listingStatus": "ACTIVE",
      "lat": 48.8584,
      "lng": 2.2945,
      "toursCount": 19,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440011",
      "code": "FR-PAR-LOUVRE",
      "name": "Louvre Museum",
      "slug": "louvre-museum",
      "listingStatus": "ACTIVE",
      "lat": 48.8606,
      "lng": 2.3376,
      "toursCount": 15,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.5 Area Ancestor/Descendant Tree (`GET /api/v1/areas/:id/tree`)

Returns the complete 4-tier ancestor and descendant tree rooted at the given area node using recursive SQL traversal.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area tree retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440008",
    "name": "France",
    "slug": "france",
    "code": "FR",
    "areaType": "COUNTRY",
    "isoCode": "FR",
    "listingStatus": "ACTIVE",
    "ancestors": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440006",
        "name": "Western Europe",
        "slug": "western-europe",
        "areaType": "SUB_CONTINENT"
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440005",
        "name": "Europe",
        "slug": "europe",
        "areaType": "CONTINENT"
      }
    ],
    "descendants": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440010",
        "name": "Eiffel Tower",
        "slug": "eiffel-tower",
        "areaType": "POI",
        "toursCount": 19
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440011",
        "name": "Louvre Museum",
        "slug": "louvre-museum",
        "areaType": "POI",
        "toursCount": 15
      }
    ]
  }
}
```

---

### 3.6 Global 4-Tier Hierarchy Tree (`GET /api/v1/areas/tree`)

Returns the complete global geographic hierarchy (Continents → Sub Continents → Countries → POIs). Cached with 24-hour TTL for high-speed navigation menus and dynamic sitemap generation.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Global area hierarchy tree retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440005",
      "name": "Europe",
      "slug": "europe",
      "areaType": "CONTINENT",
      "subContinents": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440006",
          "name": "Western Europe",
          "slug": "western-europe",
          "areaType": "SUB_CONTINENT",
          "countries": [
            {
              "id": "550e8400-e29b-41d4-a716-446655440008",
              "name": "France",
              "slug": "france",
              "code": "FR",
              "areaType": "COUNTRY",
              "pois": [
                {
                  "id": "550e8400-e29b-41d4-a716-446655440010",
                  "name": "Eiffel Tower",
                  "slug": "eiffel-tower",
                  "areaType": "POI"
                },
                {
                  "id": "550e8400-e29b-41d4-a716-446655440011",
                  "name": "Louvre Museum",
                  "slug": "louvre-museum",
                  "areaType": "POI"
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 4. Destination Landing Page Contract (`GET /api/v1/areas/:slug`)

Returns destination landing data for `/destinations/[slug]` (e.g. `/destinations/france` or `/destinations/western-europe`), including active tours count and embedded SEO metadata.

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Destination retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440008",
    "name": "France",
    "slug": "france",
    "code": "FR",
    "areaType": "COUNTRY",
    "parent": {
      "id": "550e8400-e29b-41d4-a716-446655440006",
      "name": "Western Europe",
      "slug": "western-europe"
    },
    "children": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440010",
        "name": "Eiffel Tower",
        "slug": "eiffel-tower",
        "areaType": "POI"
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440011",
        "name": "Louvre Museum",
        "slug": "louvre-museum",
        "areaType": "POI"
      }
    ],
    "activeToursCount": 22,
    "startingPrice": 28000000.00,
    "seo": {
      "metaTitle": "Paket Tour Wisata Prancis Terbaik 2026 | Hobiholidays",
      "metaDescription": "Daftar paket tour ke Prancis terlengkap. Kunjungi Menara Eiffel, Museum Louvre, dan Versailles dengan jadwal pasti di Hobiholidays.",
      "canonicalUrl": "https://www.hobiholidays.com/destinations/france",
      "ogImageUrl": "https://cdn.hobiholidays.com/areas/france/hero.jpg",
      "noIndex": false
    }
  }
}
```

### Error Response: Destination Not Found (404 Not Found)
```json
{
  "statusCode": 404,
  "message": "Destination with slug 'unknown-destination' not found",
  "error": "Not Found",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/areas/unknown-destination"
}
```

---

## 5. Admin Management Endpoints

### 5.1 Create Area (`POST /api/v1/areas`)

#### Request DTO
```typescript
import {
  IsString,
  IsUUID,
  IsInt,
  IsOptional,
  IsNumber,
  Min,
  Max,
} from 'class-validator';

export class CreateAreaDto {
  @IsInt()
  areaTypeId: number; // 1 = CONTINENT, 2 = SUB_CONTINENT, 3 = COUNTRY, 4 = POI

  @IsOptional()
  @IsUUID('4')
  parentId?: string; // Null for Continent, continentId for Sub-Continent, subContinentId for Country, countryId for POI

  @IsString()
  code: string; // e.g. "FR-PAR-EIFFEL"

  @IsString()
  name: string; // e.g. "Eiffel Tower"

  @IsString()
  slug: string; // e.g. "eiffel-tower"

  @IsOptional()
  @IsString()
  isoCode?: string; // e.g. "FR" (for countries)

  @IsOptional()
  @IsNumber()
  @Min(-90)
  @Max(90)
  lat?: number;

  @IsOptional()
  @IsNumber()
  @Min(-180)
  @Max(180)
  lng?: number;
}
```

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Area created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "parentId": "550e8400-e29b-41d4-a716-446655440008",
    "areaTypeId": 4,
    "code": "FR-PAR-EIFFEL",
    "name": "Eiffel Tower",
    "slug": "eiffel-tower",
    "listingStatus": "ACTIVE",
    "createdAt": "2026-09-04T10:00:00.000Z"
  }
}
```

---

### 5.2 Update Area (`PUT /api/v1/areas/:id`)

#### Request DTO
```typescript
import { IsString, IsOptional, IsNumber, IsIn, Min, Max } from 'class-validator';

export class UpdateAreaDto {
  @IsOptional()
  @IsString()
  name?: string;

  @IsOptional()
  @IsString()
  slug?: string;

  @IsOptional()
  @IsString()
  isoCode?: string;

  @IsOptional()
  @IsNumber()
  @Min(-90)
  @Max(90)
  lat?: number;

  @IsOptional()
  @IsNumber()
  @Min(-180)
  @Max(180)
  lng?: number;

  @IsOptional()
  @IsString()
  @IsIn(['ACTIVE', 'INACTIVE', 'ARCHIVED'])
  listingStatus?: 'ACTIVE' | 'INACTIVE' | 'ARCHIVED';

  @IsOptional()
  @IsNumber()
  sortOrder?: number;
}
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area updated successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "parentId": "550e8400-e29b-41d4-a716-446655440008",
    "areaTypeId": 4,
    "code": "FR-PAR-EIFFEL",
    "name": "Eiffel Tower",
    "slug": "eiffel-tower",
    "listingStatus": "ACTIVE",
    "updatedAt": "2026-09-04T11:00:00.000Z"
  }
}
```

---

### 5.3 Delete Area (`DELETE /api/v1/areas/:id`)

Soft deletes an area node (sets `deleted_at`). Returns 409 if the area has active child nodes or linked products.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area deleted successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "deletedAt": "2026-09-04T11:00:00.000Z"
  }
}
```

#### Error Response: Conflict (409 Conflict)
```json
{
  "statusCode": 409,
  "message": "Cannot delete area 'France': 14 active child POIs and 22 linked products exist",
  "error": "Conflict",
  "timestamp": "2026-09-04T11:00:00.000Z",
  "path": "/api/v1/areas/550e8400-e29b-41d4-a716-446655440008"
}
```
