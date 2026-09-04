# Area Domain API Contracts

> **Overview**
> API contract specifications for the Area/Geography domain, managing the **3-tier geographic hierarchy** (`Continent → Country → City`), search widget autocomplete, and destination landing page metadata.
>
> **Related Design Document:** [Area Domain Technical Design](../technical/area-technical-design.md)
> **Backend Guide:** [Area Backend Guide](../backend/area-backend-guide.md)
> **Frontend Guide:** [Area Frontend Guide](../frontend/area-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Area Types** | `GET` | `/api/v1/areas/types` | List area classifications (`CONTINENT`, `COUNTRY`, `CITY`) |
| **Hierarchy Navigation** | `GET` | `/api/v1/areas/continents` | List all continents (root nodes) |
| | `GET` | `/api/v1/areas/continents/:continentId/countries` | List countries within a continent |
| | `GET` | `/api/v1/areas/tree` | Fetch complete global 3-tier hierarchy tree (Continents → Countries → Cities) |
| | `GET` | `/api/v1/areas/:id/tree` | Fetch complete 3-tier ancestor/descendant tree rooted at node |
| **Search Widget** | `GET` | `/api/v1/areas/autocomplete` | Fast autocomplete matching continent, country, or city |
| **Destination Landing** | `GET` | `/api/v1/areas/:slug` | Destination hub page with tours count and embedded SEO |
| **Admin CRUD** | `POST` | `/api/v1/areas` | Create geographic area node |
| | `PUT` | `/api/v1/areas/:id` | Update geographic area |
| | `DELETE`| `/api/v1/areas/:id` | Soft delete area node |

---

## 1. Area Types (`GET /api/v1/areas/types`)

Returns the full list of area classifications. This is a small, static reference list — no pagination needed.

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area types retrieved successfully",
  "data": [
    {
      "id": 1,
      "name": "CONTINENT",
      "description": "Global continental landmasses and geographic macro-regions (root level)"
    },
    {
      "id": 2,
      "name": "COUNTRY",
      "description": "Sovereign states and independent nations"
    },
    {
      "id": 3,
      "name": "CITY",
      "description": "Major metropolitan areas, municipalities, and primary tour destinations (maximum granularity)"
    }
  ]
}
```

---

## 2. Search Widget Autocomplete (`GET /api/v1/areas/autocomplete`)

Powers the **"Where To?"** search widget dropdown on the homepage and All Tours navigation.

### Request Query Parameters
- `q`: Search keyword (min 2 characters, e.g. `jap` or `tokyo`)
- `limit`: Max results (default 8)

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Autocomplete results retrieved",
  "data": [
    {
      "areaId": "550e8400-e29b-41d4-a716-446655440002",
      "name": "Japan",
      "slug": "japan",
      "areaType": "COUNTRY",
      "parentName": "Asia",
      "displayName": "Japan (Asia)",
      "activeToursCount": 12
    },
    {
      "areaId": "550e8400-e29b-41d4-a716-446655440003",
      "name": "Tokyo",
      "slug": "tokyo",
      "areaType": "CITY",
      "parentName": "Japan",
      "displayName": "Tokyo, Japan",
      "activeToursCount": 8
    }
  ]
}
```

---

## 3. Hierarchy Navigation Endpoints

### 3.1 List Continents (`GET /api/v1/areas/continents`)

Returns paginated list of root-level continental nodes.

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
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "code": "ASIA",
      "name": "Asia",
      "slug": "asia",
      "listingStatus": "ACTIVE",
      "countriesCount": 18,
      "toursCount": 45,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "code": "EUROPE",
      "name": "Europe",
      "slug": "europe",
      "listingStatus": "ACTIVE",
      "countriesCount": 24,
      "toursCount": 68,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.2 List Countries in Continent (`GET /api/v1/areas/continents/:continentId/countries`)

Returns paginated list of countries under a specific continent.

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `20`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Countries retrieved successfully",
  "meta": {
    "totalItems": 18,
    "itemCount": 2,
    "itemsPerPage": 20,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "code": "JP",
      "name": "Japan",
      "slug": "japan",
      "isoCode": "JP",
      "listingStatus": "ACTIVE",
      "citiesCount": 6,
      "toursCount": 12,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440004",
      "code": "TR",
      "name": "Turkey",
      "slug": "turkey",
      "isoCode": "TR",
      "listingStatus": "ACTIVE",
      "citiesCount": 4,
      "toursCount": 9,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.3 List Cities in Country (`GET /api/v1/areas/countries/:countryId/cities`)

Returns paginated list of cities under a specific country (maximum area granularity).

#### Query Parameters
- `page` (optional, default `1`): Page number
- `limit` (optional, default `20`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Cities retrieved successfully",
  "meta": {
    "totalItems": 6,
    "itemCount": 3,
    "itemsPerPage": 20,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "code": "JP-TYO",
      "name": "Tokyo",
      "slug": "tokyo",
      "listingStatus": "ACTIVE",
      "lat": 35.6762,
      "lng": 139.6503,
      "toursCount": 8,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440005",
      "code": "JP-KYO",
      "name": "Kyoto",
      "slug": "kyoto",
      "listingStatus": "ACTIVE",
      "lat": 35.0116,
      "lng": 135.7681,
      "toursCount": 6,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440006",
      "code": "JP-OSA",
      "name": "Osaka",
      "slug": "osaka",
      "listingStatus": "ACTIVE",
      "lat": 34.6937,
      "lng": 135.5023,
      "toursCount": 5,
      "createdAt": "2026-01-01T00:00:00.000Z",
      "updatedAt": "2026-09-04T08:00:00.000Z"
    }
  ]
}
```

---

### 3.4 Area Ancestor/Descendant Tree (`GET /api/v1/areas/:id/tree`)

Returns the complete 3-tier ancestor/descendant tree rooted at the given area node.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Area tree retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "name": "Japan",
    "slug": "japan",
    "code": "JP",
    "areaType": "COUNTRY",
    "isoCode": "JP",
    "listingStatus": "ACTIVE",
    "ancestors": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "Asia",
        "slug": "asia",
        "areaType": "CONTINENT"
      }
    ],
    "descendants": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440003",
        "name": "Tokyo",
        "slug": "tokyo",
        "areaType": "CITY",
        "toursCount": 8
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440005",
        "name": "Kyoto",
        "slug": "kyoto",
        "areaType": "CITY",
        "toursCount": 6
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440006",
        "name": "Osaka",
        "slug": "osaka",
        "areaType": "CITY",
        "toursCount": 5
      }
    ]
  }
}
```

---

### 3.5 Global 3-Tier Hierarchy Tree (`GET /api/v1/areas/tree`)

Returns the complete global geographic hierarchy (all Continents nested with their Countries and Cities). Cached with 24-hour TTL for high-speed navigation menus and dynamic sitemap generation.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Global area hierarchy tree retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Asia",
      "slug": "asia",
      "areaType": "CONTINENT",
      "countries": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440002",
          "name": "Japan",
          "slug": "japan",
          "code": "JP",
          "areaType": "COUNTRY",
          "cities": [
            {
              "id": "550e8400-e29b-41d4-a716-446655440003",
              "name": "Tokyo",
              "slug": "tokyo",
              "areaType": "CITY"
            },
            {
              "id": "550e8400-e29b-41d4-a716-446655440004",
              "name": "Kyoto",
              "slug": "kyoto",
              "areaType": "CITY"
            }
          ]
        }
      ]
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "name": "Europe",
      "slug": "europe",
      "areaType": "CONTINENT",
      "countries": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440011",
          "name": "France",
          "slug": "france",
          "code": "FR",
          "areaType": "COUNTRY",
          "cities": [
            {
              "id": "550e8400-e29b-41d4-a716-446655440012",
              "name": "Paris",
              "slug": "paris",
              "areaType": "CITY"
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

Returns destination landing data for `/destinations/[slug]` (e.g. `/destinations/japan` or `/destinations/europe`), including tours count and embedded SEO metadata.

### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Destination retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "name": "Japan",
    "slug": "japan",
    "code": "JP",
    "areaType": "COUNTRY",
    "parent": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Asia",
      "slug": "asia"
    },
    "children": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440003",
        "name": "Tokyo",
        "slug": "tokyo"
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440005",
        "name": "Kyoto",
        "slug": "kyoto"
      }
    ],
    "activeToursCount": 12,
    "startingPrice": 22500000.00,
    "seo": {
      "metaTitle": "Paket Tour Wisata Jepang Terbaik & Murah 2026 | Hobiholidays",
      "metaDescription": "Daftar paket tour ke Jepang terlengkap. Kunjungi Tokyo, Kyoto, Osaka, dan Gunung Fuji dengan jadwal pasti di Hobiholidays.",
      "canonicalUrl": "https://www.hobiholidays.com/destinations/japan",
      "ogImageUrl": "https://cdn.hobiholidays.com/areas/japan/hero.jpg",
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
  areaTypeId: number; // 1 = CONTINENT, 2 = COUNTRY, 3 = CITY

  @IsOptional()
  @IsUUID('4')
  parentId?: string; // Null for continents, continentId for countries, countryId for cities

  @IsString()
  code: string; // e.g. "JP-TYO"

  @IsString()
  name: string; // e.g. "Tokyo"

  @IsString()
  slug: string; // e.g. "tokyo"

  @IsOptional()
  @IsString()
  isoCode?: string; // e.g. "JP"

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
    "id": "550e8400-e29b-41d4-a716-446655440003",
    "parentId": "550e8400-e29b-41d4-a716-446655440002",
    "areaTypeId": 3,
    "code": "JP-TYO",
    "name": "Tokyo",
    "slug": "tokyo",
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
    "id": "550e8400-e29b-41d4-a716-446655440003",
    "parentId": "550e8400-e29b-41d4-a716-446655440002",
    "areaTypeId": 3,
    "code": "JP-TYO",
    "name": "Tokyo",
    "slug": "tokyo",
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
    "id": "550e8400-e29b-41d4-a716-446655440003",
    "deletedAt": "2026-09-04T11:00:00.000Z"
  }
}
```

#### Error Response: Conflict (409 Conflict)
```json
{
  "statusCode": 409,
  "message": "Cannot delete area 'Japan': 6 active child areas and 12 linked products exist",
  "error": "Conflict",
  "timestamp": "2026-09-04T11:00:00.000Z",
  "path": "/api/v1/areas/550e8400-e29b-41d4-a716-446655440002"
}
```

