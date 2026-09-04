# Product Hierarchy API Contracts

> **Overview**
> Public-facing API contracts governing the **All Tours** listing catalog and **Variant Detail** pages. In Hobiholidays, the primary bookable unit surfaced to travelers is the **Variant (L2)**, with departures and quotas managed by **Trips (L3)** and brand context owned by **Products (L1)**.
>
> **Related Design Document:** [Product Hierarchy Technical Design](../product-hierarchy-technical-design.md)

---

## 📑 Endpoints Summary Table

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/variants` | **All Tours Catalog Feed** (returns 1 card per variant with starting price and available dates) |
| `GET` | `/api/v1/variants/:slug` | **Variant Detail Page** (aggregated public view with parent product info, active trips, itinerary brochure, and embedded SEO) |

---

## 1. All Tours Listing Feed (`GET /api/v1/variants`)

Drives the main tour package grid on the `/tours` page. Returns one listing card per active variant.

### 1.1 Query Parameters (`ListVariantsDto`)

```typescript
import { IsOptional, IsString, IsInt, Min, IsIn } from 'class-validator';
import { Type } from 'class-transformer';

export class ListVariantsDto {
  @IsOptional()
  @IsString()
  @IsIn(['STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL'])
  variantType?: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';

  @IsOptional()
  @IsString()
  @IsIn(['ALL', 'DOMESTIC', 'INTERNATIONAL'])
  nationalityScope?: 'ALL' | 'DOMESTIC' | 'INTERNATIONAL' = 'ALL';

  @IsOptional()
  @IsString()
  @IsIn(['price_asc', 'price_desc', 'newest', 'duration_asc', 'duration_desc'])
  sortBy?: string = 'price_asc';

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  limit?: number = 12;
}
```

### 1.2 Success Response (200 OK)

```json
{
  "statusCode": 200,
  "message": "Variants retrieved successfully",
  "meta": {
    "totalItems": 24,
    "itemCount": 2,
    "itemsPerPage": 12,
    "totalPages": 2,
    "currentPage": 1
  },
  "data": [
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "code": "GWE-SPR-2026",
      "name": "GWE Spring 2026",
      "slug": "gwe-spring-2026",
      "variantType": "SEASONAL",
      "badge": "🌸 Spring Edition",
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "durationDays": 11,
      "durationNights": 9,
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "city": "Amsterdam",
          "country": "Netherlands",
          "continent": "Europe"
        },
        {
          "city": "Paris",
          "country": "France",
          "continent": "Europe"
        }
      ],
      "startingPrice": 28000000.00,
      "currency": "IDR",
      "nextDepartureDate": "2026-04-10",
      "totalActiveDepartures": 4
    },
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440021",
      "code": "GWE-TLP-2026",
      "name": "Tulip Keukenhof Special",
      "slug": "tulip-keukenhof-special",
      "variantType": "THEMED",
      "badge": "🌷 Keukenhof Special",
      "productId": "550e8400-e29b-41d4-a716-446655440010",
      "productName": "Grand West Europe",
      "productSlug": "grand-west-europe",
      "durationDays": 9,
      "durationNights": 7,
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "city": "Amsterdam",
          "country": "Netherlands",
          "continent": "Europe"
        },
        {
          "city": "Brussels",
          "country": "Belgium",
          "continent": "Europe"
        }
      ],
      "startingPrice": 31000000.00,
      "currency": "IDR",
      "nextDepartureDate": "2026-05-02",
      "totalActiveDepartures": 2
    }
  ]
}
```

---

## 2. Variant Detail Page Contract (`GET /api/v1/variants/:slug`)

Returns the aggregated payload required to render the full tour detail page (`/tours/[productSlug]/[variantSlug]`), including upcoming dated departures, itinerary timeline, downloadable brochure, and embedded SEO metadata.

### 2.1 Success Response (200 OK)

```json
{
  "statusCode": 200,
  "message": "Variant detail retrieved successfully",
  "data": {
    "variant": {
      "id": "550e8400-e29b-41d4-a716-446655440020",
      "code": "GWE-SPR-2026",
      "name": "GWE Spring 2026",
      "slug": "gwe-spring-2026",
      "variantType": "SEASONAL",
      "listingStatus": "ACTIVE",
      "durationDays": 11,
      "durationNights": 9,
      "startingPrice": 28000000.00,
      "currency": "IDR"
    },
    "product": {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "name": "Grand West Europe",
      "slug": "grand-west-europe",
      "productType": "JOURNEY",
      "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf"
    },
    "media": {
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "gallery": [
        "https://cdn.hobiholidays.com/products/gwe/amsterdam-canals.jpg",
        "https://cdn.hobiholidays.com/products/gwe/brussels-atomium.jpg",
        "https://cdn.hobiholidays.com/products/gwe/paris-louvre.jpg"
      ]
    },
    "destinations": [
      {
        "city": "Amsterdam",
        "country": "Netherlands",
        "countryCode": "NL",
        "continent": "Europe"
      },
      {
        "city": "Brussels",
        "country": "Belgium",
        "countryCode": "BE",
        "continent": "Europe"
      },
      {
        "city": "Paris",
        "country": "France",
        "countryCode": "FR",
        "continent": "Europe"
      }
    ],
    "upcomingTrips": [
      {
        "tripId": "550e8400-e29b-41d4-a716-446655440031",
        "startDate": "2026-04-10",
        "endDate": "2026-04-20",
        "minQuota": 1,
        "maxQuota": 25,
        "availableSeats": 8,
        "status": "ACTIVE",
        "prices": {
          "sellingPrice": 28000000.00,
          "basePrice": 32000000.00,
          "nationalityScope": "ALL"
        }
      },
      {
        "tripId": "550e8400-e29b-41d4-a716-446655440032",
        "startDate": "2026-04-24",
        "endDate": "2026-05-04",
        "minQuota": 1,
        "maxQuota": 25,
        "availableSeats": 14,
        "status": "ACTIVE",
        "prices": {
          "sellingPrice": 28000000.00,
          "basePrice": 32000000.00,
          "nationalityScope": "ALL"
        }
      }
    ],
    "seo": {
      "metaTitle": "Tour GWE Spring 2026 (11D/9N) Eropa Barat Murah | Hobiholidays",
      "metaDescription": "Nikmati keindahan musim semi di Belanda, Belgia, dan Prancis bersama paket tour GWE Spring 2026. Keberangkatan April 2026.",
      "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe/gwe-spring-2026",
      "ogImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "noIndex": false
    }
  }
}
```

### 2.2 Error Response: Variant Not Found (404 Not Found)

```json
{
  "statusCode": 404,
  "message": "Variant with slug 'unknown-variant' not found",
  "error": "Not Found",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/variants/unknown-variant"
}
```
