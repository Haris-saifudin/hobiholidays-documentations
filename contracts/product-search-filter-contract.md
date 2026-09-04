# Search & Filter API Contracts

> **Overview**
> Comprehensive API contract specification for the Search & Filter engine (`GET /api/v1/variants/search`). This contract maps frontend search widgets, All Tours multi-attribute filters, and pagination parameters to validated NestJS DTOs and structured JSON responses.
>
> **Related Design Document:** [Product Search & Filter Architecture](../technical/product-search-filter-technical-design.md)
> **Backend Guide:** [Search & Filter Backend Guide](../backend/product-search-filter-backend-guide.md)
> **Frontend Guide:** [Search & Filter Frontend Guide](../frontend/product-search-filter-frontend-guide.md)

---

## 📑 Endpoint Specification

- **Method:** `GET`
- **Path:** `/api/v1/variants/search`
- **Authentication:** Public (No bearer token required)
- **Cache Policy:** `Cache-Control: public, s-maxage=60, stale-while-revalidate=300`

---

## 1. Request Query Parameters (`SearchTripDto`)

The backend validates incoming query string parameters using NestJS `class-validator` and `class-transformer`.

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

  @IsOptional()
  @IsString()
  continentSlug?: string; // e.g. "europe", "asia"

  @IsOptional()
  @IsString()
  continentId?: string; // UUID v4

  @IsOptional()
  @IsString()
  countrySlug?: string; // e.g. "japan", "turkey", "netherlands"

  @IsOptional()
  @IsString()
  countryId?: string; // UUID v4

  @IsOptional()
  @IsString()
  destination?: string; // Free text search matching continent, country, or city

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
  @IsString()
  @IsIn(['ALL', 'DOMESTIC', 'INTERNATIONAL'])
  nationalityScope?: 'ALL' | 'DOMESTIC' | 'INTERNATIONAL' = 'ALL';

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
  country: string;
  city: string;
}

export interface SearchVariantCard {
  variantId: string;
  variantName: string;
  variantSlug: string;
  variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';
  productId: string;
  productName: string;
  productSlug: string;
  durationDays: number;
  durationNights: number;
  coverImageUrl: string;
  destinations: DestinationHierarchy[];
  availableDates: string[]; // ISO Date string: "YYYY-MM-DD"
  startingPrice: number;    // Lowest selling_price for matching trips
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
      "durationDays": 11,
      "durationNights": 9,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "country": "Netherlands",
          "city": "Amsterdam"
        },
        {
          "continent": "Europe",
          "country": "France",
          "city": "Paris"
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
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "country": "Netherlands",
          "city": "Amsterdam"
        },
        {
          "continent": "Europe",
          "country": "Belgium",
          "city": "Brussels"
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
      "durationDays": 9,
      "durationNights": 7,
      "coverImageUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "continent": "Europe",
          "country": "Netherlands",
          "city": "Amsterdam"
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

## 4. Error Responses

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
