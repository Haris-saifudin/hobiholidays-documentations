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

  // Multi-Dimensional Category filters
  @IsOptional()
  @IsString({ each: true })
  categorySlugs?: string[]; // Array or comma-separated slugs e.g. ['classic-series', 'spring-blossom']

  @IsOptional()
  @IsString({ each: true })
  categoryIds?: string[]; // Array of UUID v4

  @IsOptional()
  @IsString()
  travelStyleSlug?: string; // Dimension: TRAVEL_STYLE (e.g. "classic-series", "luxury-escapes")

  @IsOptional()
  @IsString()
  themeSlug?: string; // Dimension: THEME_INTEREST (e.g. "heritage-culture", "flower-season")

  @IsOptional()
  @IsString()
  seasonSlug?: string; // Dimension: SEASON_MOMENT (e.g. "spring-blossom", "autumn-foliage")

  @IsOptional()
  @IsString()
  specialSlug?: string; // Dimension: SPECIAL_EXPERIENCE (e.g. "scenic-trains", "culinary-masterclass")

  // Promotional Badges
  @IsOptional()
  @IsString()
  badgeCode?: string; // e.g. "PASTI_BERANGKAT", "EARLY_BIRD"

  @IsOptional()
  @IsString({ each: true })
  badgeCodes?: string[]; // Array of badge codes e.g. ["PASTI_BERANGKAT", "HOT_DEAL"]

  // Duration Bracket
  @IsOptional()
  @IsString()
  durationBracket?: '1-3' | '4-7' | '8-14' | '15+'; // Duration bracket filter (e.g. "4-7")

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
export interface DestinationHierarchyDto {
  continent: string;              // Always resolved (Root)
  continentSlug?: string;
  subContinent?: string | null;   // Null when anchored directly to CONTINENT
  subContinentSlug?: string | null;
  country?: string | null;        // Null when anchored to CONTINENT or SUB_CONTINENT
  countrySlug?: string | null;
  poi?: string | null;            // Null when anchored to CONTINENT, SUB_CONTINENT, or COUNTRY
  poiSlug?: string | null;
}

export type DestinationHierarchy = DestinationHierarchyDto;

export interface CategoryAssignmentDto {
  dimension: 'TRAVEL_STYLE' | 'THEME_INTEREST' | 'SEASON_MOMENT' | 'SPECIAL_EXPERIENCE';
  dimensionName: string;
  id: string;
  name: string;
  slug: string;
}

export interface VariantBadgeDto {
  id: string;
  code: string;
  label: string;
  backgroundColor: string;
  textColor: string;
  iconUrl?: string | null;
}

export interface SearchVariantCard {
  variantId: string;
  variantName: string;
  variantSlug: string;
  variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';
  badges: VariantBadgeDto[];
  productId: string;
  productName: string;
  productSlug: string;
  categories: CategoryAssignmentDto[];
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
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440091",
          "code": "PASTI_BERANGKAT",
          "label": "⚡ Pasti Berangkat",
          "backgroundColor": "#ECFDF5",
          "textColor": "#065F46",
          "iconUrl": null
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440093",
          "code": "HOT_DEAL",
          "label": "🔥 Hot Deal",
          "backgroundColor": "#FEF2F2",
          "textColor": "#991B1B",
          "iconUrl": null
        }
      ],
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "categories": [
        {
          "dimension": "TRAVEL_STYLE",
          "dimensionName": "Travel Style",
          "id": "550e8400-e29b-41d4-a716-446655440081",
          "name": "Classic Series",
          "slug": "classic-series"
        },
        {
          "dimension": "SEASON_MOMENT",
          "dimensionName": "Season & Moment",
          "id": "550e8400-e29b-41d4-a716-446655440083",
          "name": "Spring Cherry Blossom",
          "slug": "spring-blossom"
        }
      ],
      "durationDays": 7,
      "durationNights": 6,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "continentSlug": "europe",
          "subContinent": "Western Europe",
          "subContinentSlug": "western-europe",
          "country": "Netherlands",
          "countrySlug": "netherlands",
          "poi": "Keukenhof",
          "poiSlug": "keukenhof"
        },
        {
          "continent": "Europe",
          "continentSlug": "europe",
          "subContinent": "Western Europe",
          "subContinentSlug": "western-europe",
          "country": "France",
          "countrySlug": "france",
          "poi": "Eiffel Tower",
          "poiSlug": "eiffel-tower"
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
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440092",
          "code": "EARLY_BIRD",
          "label": "🎟️ Early Bird Promo",
          "backgroundColor": "#EFF6FF",
          "textColor": "#1E40AF",
          "iconUrl": null
        }
      ],
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "categories": [
        {
          "dimension": "TRAVEL_STYLE",
          "dimensionName": "Travel Style",
          "id": "550e8400-e29b-41d4-a716-446655440081",
          "name": "Classic Series",
          "slug": "classic-series"
        },
        {
          "dimension": "THEME_INTEREST",
          "dimensionName": "Theme & Interest",
          "id": "550e8400-e29b-41d4-a716-446655440082",
          "name": "Flower Season",
          "slug": "flower-season"
        }
      ],
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "continentSlug": "europe",
          "subContinent": "Western Europe",
          "subContinentSlug": "western-europe",
          "country": "Netherlands",
          "countrySlug": "netherlands",
          "poi": "Keukenhof",
          "poiSlug": "keukenhof"
        },
        {
          "continent": "Europe",
          "continentSlug": "europe",
          "subContinent": "Western Europe",
          "subContinentSlug": "western-europe",
          "country": "Belgium",
          "countrySlug": "belgium",
          "poi": "Grand Place",
          "poiSlug": "grand-place"
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
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440092",
          "code": "EARLY_BIRD",
          "label": "🎟️ Early Bird Promo",
          "backgroundColor": "#EFF6FF",
          "textColor": "#1E40AF",
          "iconUrl": null
        }
      ],
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "categories": [
        {
          "dimension": "TRAVEL_STYLE",
          "dimensionName": "Travel Style",
          "id": "550e8400-e29b-41d4-a716-446655440081",
          "name": "Classic Series",
          "slug": "classic-series"
        },
        {
          "dimension": "THEME_INTEREST",
          "dimensionName": "Theme & Interest",
          "id": "550e8400-e29b-41d4-a716-446655440082",
          "name": "Flower Season",
          "slug": "flower-season"
        }
      ],
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "continentSlug": "europe",
          "subContinent": "Western Europe",
          "subContinentSlug": "western-europe",
          "country": "Netherlands",
          "countrySlug": "netherlands",
          "poi": "Keukenhof",
          "poiSlug": "keukenhof"
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

export interface DestinationSubContinentOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  areaType: 'SUB_CONTINENT';
  activePackagesCount: number;
  countries: DestinationCountryOption[];
}

export interface DestinationContinentOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  areaType: 'CONTINENT';
  activePackagesCount: number;
  subContinents: DestinationSubContinentOption[];
}

export interface CategoryChildOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  activePackagesCount: number;
}

export interface CategoryOption {
  id: string; // UUID v4
  name: string;
  slug: string;
  activePackagesCount: number;
  children?: CategoryChildOption[];
}

export interface CategoryDimensionOption {
  id: string; // UUID v4
  code: 'TRAVEL_STYLE' | 'THEME_INTEREST' | 'SEASON_MOMENT' | 'SPECIAL_EXPERIENCE';
  name: string;
  categories: CategoryOption[];
}

export interface BadgeOption {
  id: string; // UUID v4
  code: string;
  label: string;
  backgroundColor: string;
  textColor: string;
  activePackagesCount: number;
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

export interface DurationBracketOption {
  key: '1-3' | '4-7' | '8-14' | '15+';
  label: string;
  count: number;
}

export interface FilterOptionsResponseDto {
  destinations: DestinationContinentOption[];
  categories: CategoryDimensionOption[];
  badges: BadgeOption[];
  priceRange: PriceRangeOption;
  departureMonths: DepartureMonthOption[];
  durationBrackets: DurationBracketOption[];
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
        "subContinents": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440002",
            "name": "Western Europe",
            "slug": "western-europe",
            "areaType": "SUB_CONTINENT",
            "activePackagesCount": 10,
            "countries": [
              {
                "id": "550e8400-e29b-41d4-a716-446655440003",
                "name": "Netherlands",
                "slug": "netherlands",
                "areaType": "COUNTRY",
                "activePackagesCount": 6,
                "pois": [
                  {
                    "id": "550e8400-e29b-41d4-a716-446655440004",
                    "name": "Keukenhof Gardens",
                    "slug": "keukenhof",
                    "areaType": "POI",
                    "activePackagesCount": 4
                  }
                ]
              },
              {
                "id": "550e8400-e29b-41d4-a716-446655440005",
                "name": "France",
                "slug": "france",
                "areaType": "COUNTRY",
                "activePackagesCount": 4,
                "pois": [
                  {
                    "id": "550e8400-e29b-41d4-a716-446655440006",
                    "name": "Eiffel Tower",
                    "slug": "eiffel-tower",
                    "areaType": "POI",
                    "activePackagesCount": 4
                  }
                ]
              }
            ]
          }
        ]
      }
    ],
    "categories": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440071",
        "code": "TRAVEL_STYLE",
        "name": "Travel Style",
        "categories": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440081",
            "name": "Classic Series",
            "slug": "classic-series",
            "activePackagesCount": 8
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440082",
            "name": "Luxury Escapes",
            "slug": "luxury-escapes",
            "activePackagesCount": 3
          }
        ]
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440072",
        "code": "THEME_INTEREST",
        "name": "Theme & Interest",
        "categories": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440083",
            "name": "Heritage & Culture",
            "slug": "heritage-culture",
            "activePackagesCount": 7
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440084",
            "name": "Flower Season",
            "slug": "flower-season",
            "activePackagesCount": 6
          }
        ]
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440073",
        "code": "SEASON_MOMENT",
        "name": "Season & Moment",
        "categories": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440085",
            "name": "Spring Cherry Blossom",
            "slug": "spring-blossom",
            "activePackagesCount": 5
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440086",
            "name": "Autumn Foliage",
            "slug": "autumn-foliage",
            "activePackagesCount": 4
          }
        ]
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440074",
        "code": "SPECIAL_EXPERIENCE",
        "name": "Special Experience",
        "categories": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440087",
            "name": "Scenic Train Rides",
            "slug": "scenic-trains",
            "activePackagesCount": 3
          }
        ]
      }
    ],
    "badges": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440091",
        "code": "PASTI_BERANGKAT",
        "label": "⚡ Pasti Berangkat",
        "backgroundColor": "#ECFDF5",
        "textColor": "#065F46",
        "activePackagesCount": 6
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440092",
        "code": "EARLY_BIRD",
        "label": "🎟️ Early Bird Promo",
        "backgroundColor": "#EFF6FF",
        "textColor": "#1E40AF",
        "activePackagesCount": 4
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
    "durationBrackets": [
      {
        "key": "4-7",
        "label": "4 - 7 Hari",
        "count": 5
      },
      {
        "key": "8-14",
        "label": "8 - 14 Hari",
        "count": 9
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
