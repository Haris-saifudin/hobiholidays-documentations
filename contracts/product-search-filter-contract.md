# Search & Filter API Contracts

> **Overview**
> Comprehensive API contract specification for the Search & Filter engine (`GET /api/v1/variants/search`). This contract maps frontend search widgets, All Tours multi-attribute filters, and pagination parameters to validated NestJS DTOs and structured JSON responses.
>
> **Related Design Document:** [Product Search & Filter Architecture](../technical/product-search-filter-technical-design.md)  
> **Backend Guide:** [Search & Filter Backend Guide](../backend/product-search-filter-backend-guide.md)  
> **Frontend Guide:** [Search & Filter Frontend Guide](../frontend/product-search-filter-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Filter Master Data** | `GET` | `/api/v1/variants/search/filter-options` | Retrieve dynamic master data filter options (active destinations, categories, price range, departure months) |
| **Catalog Search** | `GET` | `/api/v1/variants/search` | Multi-attribute catalog search query returning paginated variant cards |

---

## 1. Catalog Search Endpoint (`GET /api/v1/variants/search`)

- **Method:** `GET`
- **Path:** `/api/v1/variants/search`
- **Authentication:** Public (No bearer token required)
- **Cache Policy:** `Cache-Control: public, s-maxage=60, stale-while-revalidate=300`

### 1.1 Request Query Parameters (`SearchTripDto`)

The backend validates incoming query string parameters using NestJS `class-validator` and `class-transformer`. Note that pricing is evaluated on the standard `ADULT` age band (`nationality_scope` is completely removed).

```typescript
// search-trip.dto.ts
import {
  IsOptional,
  IsString,
  IsInt,
  IsNumber,
  Min,
  IsIn,
  Matches,
} from 'class-validator';
import { Type } from 'class-transformer';

export class SearchTripDto {
  @IsOptional()
  @IsString()
  productName?: string; // e.g. "Grand West Europe", "Tulip"

  // 2-tier Category filters
  @IsOptional()
  @IsString()
  parentCategorySlug?: string; // e.g. "tour-series", "special-interest"

  @IsOptional()
  @IsString()
  parentCategoryId?: string; // UUID v4

  @IsOptional()
  @IsString()
  categorySlug?: string; // e.g. "classic-series", "flower-season"

  @IsOptional()
  @IsString()
  categoryId?: string; // UUID v4

  // 4-tier Area filters
  @IsOptional()
  @IsString()
  continentSlug?: string; // e.g. "europe", "asia"

  @IsOptional()
  @IsString()
  continentId?: string; // UUID v4

  @IsOptional()
  @IsString()
  subContinentSlug?: string; // e.g. "western-europe", "east-asia"

  @IsOptional()
  @IsString()
  subContinentId?: string; // UUID v4

  @IsOptional()
  @IsString()
  countrySlug?: string; // e.g. "france", "netherlands", "japan"

  @IsOptional()
  @IsString()
  countryId?: string; // UUID v4

  @IsOptional()
  @IsString()
  poiSlug?: string; // e.g. "eiffel-tower", "keukenhof"

  @IsOptional()
  @IsString()
  poiId?: string; // UUID v4

  @IsOptional()
  @IsString()
  destination?: string; // Free text search matching continent, sub-continent, country, or POI

  @IsOptional()
  @IsString()
  @Matches(/^\d{4}-(0[1-9]|1[0-2])$/, {
    message: 'departureMonth must be in YYYY-MM format (e.g. 2026-10)',
  })
  departureMonth?: string;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  totalPack?: number = 1; // Total passengers / travel party size

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  pax?: number; // Backward-compatible alias for totalPack

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  minPrice?: number; // Minimum price filter in IDR (e.g. 20000000)

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(0)
  maxPrice?: number; // Maximum price filter in IDR (e.g. 35000000)

  @IsOptional()
  @IsString()
  @IsIn(['STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL'])
  variantType?: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  limit?: number = 10;
}
```

---

## 2. Response Contracts

### 2.1 TypeScript Response Interfaces

```typescript
// search-trip-response.interface.ts
export interface DestinationHierarchy {
  continent: string;
  subContinent?: string;
  country: string;
  poi?: string;
}

export interface CategorySummary {
  id: string;
  name: string;
  slug: string;
}

export interface SearchVariantCard {
  variantId: string;
  variantName: string;
  variantSlug: string;
  variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';
  productId: string;
  productName: string;
  productSlug: string;
  category: CategorySummary;
  parentCategory: CategorySummary;
  durationDays: number;
  durationNights: number;
  coverImageUrl: string;
  destinations: DestinationHierarchy[];
  availableDates: string[]; // ISO Date string: "YYYY-MM-DD"
  startingPrice: number;    // Lowest selling_price for ADULT age band
  currency: string;         // "IDR"
}

export interface SearchTripResponse {
  statusCode: number;
  message: string;
  meta: {
    totalItems: number;     // Total matching records
    itemCount: number;      // Records on current page
    itemsPerPage: number;   // Limit per page
    totalPages: number;     // Total pages available
    currentPage: number;    // Active page (1-indexed)
    totalPackages: number;  // Alias for totalItems — used in UI badges ("Found X Tour Packages")
  };
  data: SearchVariantCard[];
}
```

---

## 3. Real-World Request & Response Scenarios

### Scenario 1: Continent + Budget Range + Total Pack Filter
A user selects **Europe** as continent, sets budget between **IDR 25M and 35M**, for **2 Packs**.

#### HTTP Request
```http
GET /api/v1/variants/search?continentSlug=europe&minPrice=25000000&maxPrice=35000000&totalPack=2&page=1&limit=10 HTTP/1.1
Host: api.hobiholidays.com
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Search completed successfully",
  "meta": {
    "totalItems": 2,
    "itemCount": 2,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1,
    "totalPackages": 2
  },
  "data": [
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "variantName": "GWE Spring 2026",
      "variantSlug": "gwe-spring-2026",
      "variantType": "SEASONAL",
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "category": {
        "id": "550e8400-e29b-41d4-a716-446655440081",
        "name": "Classic Series",
        "slug": "classic-series"
      },
      "parentCategory": {
        "id": "550e8400-e29b-41d4-a716-446655440080",
        "name": "Tour Series",
        "slug": "tour-series"
      },
      "durationDays": 11,
      "durationNights": 9,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "subContinent": "Western Europe",
          "country": "Netherlands",
          "poi": "Keukenhof"
        },
        {
          "continent": "Europe",
          "subContinent": "Western Europe",
          "country": "France",
          "poi": "Eiffel Tower"
        }
      ],
      "availableDates": [
        "2026-04-10",
        "2026-04-24"
      ],
      "startingPrice": 28000000.00,
      "currency": "IDR"
    },
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440021",
      "variantName": "Tulip Keukenhof Special",
      "variantSlug": "tulip-keukenhof-special",
      "variantType": "THEMED",
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "category": {
        "id": "550e8400-e29b-41d4-a716-446655440082",
        "name": "Flower Season",
        "slug": "flower-season"
      },
      "parentCategory": {
        "id": "550e8400-e29b-41d4-a716-446655440080",
        "name": "Tour Series",
        "slug": "tour-series"
      },
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "subContinent": "Western Europe",
          "country": "Netherlands",
          "poi": "Keukenhof"
        },
        {
          "continent": "Europe",
          "subContinent": "Western Europe",
          "country": "Belgium",
          "poi": "Grand Place"
        }
      ],
      "availableDates": [
        "2026-05-02"
      ],
      "startingPrice": 31000000.00,
      "currency": "IDR"
    }
  ]
}
```

---

### Scenario 2: Search by Product Name
A user types `"Tulip"` in the tour search bar.

#### HTTP Request
```http
GET /api/v1/variants/search?productName=Tulip HTTP/1.1
Host: api.hobiholidays.com
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Search completed successfully",
  "meta": {
    "totalItems": 1,
    "itemCount": 1,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1,
    "totalPackages": 1
  },
  "data": [
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440021",
      "variantName": "Tulip Keukenhof Special",
      "variantSlug": "tulip-keukenhof-special",
      "variantType": "THEMED",
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "category": {
        "id": "550e8400-e29b-41d4-a716-446655440082",
        "name": "Flower Season",
        "slug": "flower-season"
      },
      "parentCategory": {
        "id": "550e8400-e29b-41d4-a716-446655440080",
        "name": "Tour Series",
        "slug": "tour-series"
      },
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "subContinent": "Western Europe",
          "country": "Netherlands",
          "poi": "Keukenhof"
        }
      ],
      "availableDates": [
        "2026-05-02"
      ],
      "startingPrice": 31000000.00,
      "currency": "IDR"
    }
  ]
}
```

---

---

## 4. Filter Options Master Data Endpoint (`GET /api/v1/variants/search/filter-options`)

Serves dynamic, aggregated master data options for populating search filter dropdowns and the "Where To?" search widget. Only destinations, categories, and departure dates that have **active tours (`activePackagesCount > 0`)** are returned, preventing zero-result queries.

- **Method:** `GET`
- **Path:** `/api/v1/variants/search/filter-options`
- **Authentication:** Public (No bearer token required)
- **Cache Policy:** `Cache-Control: public, s-maxage=300, stale-while-revalidate=600` (backed by Redis 1-hour TTL)

### 4.1 Response DTO Interface (`FilterOptionsResponseDto`)

```typescript
export interface DestinationPoiOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  areaType: 'POI';
  activePackagesCount: number;
}

export interface DestinationCountryOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  areaType: 'COUNTRY';
  activePackagesCount: number;
  pois: DestinationPoiOption[];
}

export interface DestinationContinentOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  areaType: 'CONTINENT';
  activePackagesCount: number;
  countries: DestinationCountryOption[];
}

export interface CategoryChildOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  activePackagesCount: number;
}

export interface CategoryParentOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  children: CategoryChildOption[];
}

export interface PriceRangeOption {
  currency: string;
  min: number;
  max: number;
}

export interface DepartureMonthOption {
  value: string; // "YYYY-MM"
  label: string; // "August 2026"
  activeTripsCount: number;
}

export interface VariantTypeOption {
  key: string;   // "STANDARD" | "SEASONAL" | "THEMED" | "PROMOTIONAL"
  label: string; // "Standard All-Year"
  count: number;
}

export interface FilterOptionsResponseDto {
  destinations: DestinationContinentOption[];
  categories: CategoryParentOption[];
  priceRange: PriceRangeOption;
  departureMonths: DepartureMonthOption[];
  variantTypes: VariantTypeOption[];
}
```

### 4.2 Success Response (200 OK)

```json
{
  "statusCode": 200,
  "message": "Filter options retrieved successfully",
  "data": {
    "destinations": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "name": "Europe",
        "slug": "europe",
        "areaType": "CONTINENT",
        "activePackagesCount": 14,
        "countries": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440002",
            "name": "Netherlands",
            "slug": "netherlands",
            "areaType": "COUNTRY",
            "activePackagesCount": 6,
            "pois": [
              {
                "id": "550e8400-e29b-41d4-a716-446655440003",
                "name": "Keukenhof Gardens",
                "slug": "keukenhof",
                "areaType": "POI",
                "activePackagesCount": 4
              }
            ]
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440004",
            "name": "Turkey",
            "slug": "turkey",
            "areaType": "COUNTRY",
            "activePackagesCount": 8,
            "pois": [
              {
                "id": "550e8400-e29b-41d4-a716-446655440005",
                "name": "Cappadocia",
                "slug": "cappadocia",
                "areaType": "POI",
                "activePackagesCount": 5
              }
            ]
          }
        ]
      }
    ],
    "categories": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440010",
        "name": "Tour Series",
        "slug": "tour-series",
        "children": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440011",
            "name": "Classic Series",
            "slug": "classic-series",
            "activePackagesCount": 8
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440012",
            "name": "Flower Season",
            "slug": "flower-season",
            "activePackagesCount": 6
          }
        ]
      }
    ],
    "priceRange": {
      "currency": "IDR",
      "min": 18500000.00,
      "max": 48000000.00
    },
    "departureMonths": [
      {
        "value": "2026-08",
        "label": "August 2026",
        "activeTripsCount": 5
      },
      {
        "value": "2026-09",
        "label": "September 2026",
        "activeTripsCount": 9
      },
      {
        "value": "2026-10",
        "label": "October 2026",
        "activeTripsCount": 4
      }
    ],
    "variantTypes": [
      {
        "key": "STANDARD",
        "label": "Standard All-Year",
        "count": 6
      },
      {
        "key": "SEASONAL",
        "label": "Seasonal Edition",
        "count": 5
      },
      {
        "key": "THEMED",
        "label": "Themed Edition",
        "count": 3
      }
    ]
  }
}
```

---

## 5. Error Responses

### Validation Error (400 Bad Request)
Emitted when query parameter validation fails:

```json
{
  "statusCode": 400,
  "message": [
    "departureMonth must be in YYYY-MM format (e.g. 2026-10)",
    "minPrice must not be less than 0"
  ],
  "error": "Bad Request",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/variants/search"
}
```
