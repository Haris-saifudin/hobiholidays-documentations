# Product Hierarchy API Contracts

> **Overview**
> Public-facing API contracts governing the **All Tours** listing catalog and **Variant Detail** pages. In Hobiholidays, the primary bookable unit surfaced to travelers is the **Variant (L2)**, with departures and quotas managed by **Trips (L3)** and brand context owned by **Products (L1)**.
>
> **Related Design Document:** [Product Hierarchy Technical Design](../technical/product-hierarchy-technical-design.md)  
> **Backend Guide:** [Product Hierarchy Backend Guide](../backend/product-hierarchy-backend-guide.md)  
> **Frontend Guide:** [Product Hierarchy Frontend Guide](../frontend/product-hierarchy-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/api/v1/variants` | **All Tours Catalog Feed** (returns 1 card per variant with category badges, 4-tier destinations, and Adult starting price) |
| `GET` | `/api/v1/variants/:slug` | **Variant Detail Page** (aggregated public view with parent product info, default master itinerary, add-ons, active trips, age-band pricing breakdowns, and embedded SEO) |

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
  parentCategorySlug?: string; // e.g. "tour-series"

  @IsOptional()
  @IsString()
  categorySlug?: string; // e.g. "classic-series"

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
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440091",
          "code": "SPRING_EDITION",
          "label": "🌸 Spring Edition",
          "backgroundColor": "#FDF2F8",
          "textColor": "#9D174D"
        }
      ],
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
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "poi": "Keukenhof",
          "country": "Netherlands",
          "subContinent": "Western Europe",
          "continent": "Europe"
        },
        {
          "poi": "Eiffel Tower",
          "country": "France",
          "subContinent": "Western Europe",
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
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440093",
          "code": "TULIP_SPECIAL",
          "label": "🌷 Keukenhof Special",
          "backgroundColor": "#F0FDF4",
          "textColor": "#166534"
        }
      ],
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
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "poi": "Keukenhof",
          "country": "Netherlands",
          "subContinent": "Western Europe",
          "continent": "Europe"
        },
        {
          "poi": "Grand Place",
          "country": "Belgium",
          "subContinent": "Western Europe",
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

Returns the aggregated payload required to render the full tour detail page (`/tours/[productSlug]/[variantSlug]`), including upcoming dated departures, age-band pricing breakdowns with itemized components, variant default master itinerary, optional add-ons, and embedded SEO metadata.

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
      "currency": "IDR",
      "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-spring-2026-brochure.pdf",
      "badges": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440091",
          "code": "SPRING_EDITION",
          "label": "🌸 Spring Edition",
          "backgroundColor": "#FDF2F8",
          "textColor": "#9D174D"
        }
      ]
    },
    "product": {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "name": "Grand West Europe",
      "slug": "grand-west-europe",
      "productType": "JOURNEY",
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
        "poi": "Keukenhof",
        "country": "Netherlands",
        "countryCode": "NL",
        "subContinent": "Western Europe",
        "continent": "Europe"
      },
      {
        "poi": "Grand Place",
        "country": "Belgium",
        "countryCode": "BE",
        "subContinent": "Western Europe",
        "continent": "Europe"
      },
      {
        "poi": "Eiffel Tower",
        "country": "France",
        "countryCode": "FR",
        "subContinent": "Western Europe",
        "continent": "Europe"
      }
    ],
    "itinerary": {
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "title": "Master Default Itinerary 11D/9N",
      "days": [
        {
          "dayNumber": 1,
          "sequenceNumber": 1,
          "itemType": "TRANSPORT",
          "title": "Jakarta - Doha - Amsterdam",
          "description": "Boarding flight to Amsterdam via Doha.",
          "meals": "Meals on Board"
        },
        {
          "dayNumber": 2,
          "sequenceNumber": 1,
          "itemType": "ACTIVITY",
          "title": "Amsterdam - Keukenhof - Zaanse Schans",
          "description": "Explore Keukenhof tulip garden and windmill village.",
          "meals": "Breakfast, Dinner"
        },
        {
          "dayNumber": 3,
          "sequenceNumber": 1,
          "itemType": "OTHER",
          "title": "Amsterdam Free Day & Canal Leisure",
          "description": "Free time for self-guided exploration, shopping at Dam Square, or canal walking.",
          "meals": "Breakfast"
        }
      ]
    },
    "addons": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440070",
        "code": "ADDON-VISA-FAST",
        "name": "Schengen Visa Fast Track",
        "addonType": "VISA_EXPRESS",
        "chargeType": "PER_PAX",
        "price": 2500000.00,
        "currency": "IDR",
        "applicableAgeBand": null,
        "isMandatory": false,
        "maxQuantity": 1
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440071",
        "code": "ADDON-SINGLE-SUPP",
        "name": "Single Supplement (Kamar Sendiri)",
        "addonType": "SINGLE_ROOM",
        "chargeType": "PER_ROOM",
        "price": 8500000.00,
        "currency": "IDR",
        "applicableAgeBand": "ADULT",
        "isMandatory": false,
        "maxQuantity": 1
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
        "hasTripItineraryOverride": false,
        "pricings": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440041",
            "ageBand": "ADULT",
            "ageMin": 12,
            "ageMax": null,
            "consumesQuota": true,
            "basePrice": 32000000.00,
            "sellingPrice": 28000000.00
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440044",
            "ageBand": "INFANT",
            "ageMin": 0,
            "ageMax": 2,
            "consumesQuota": false,
            "basePrice": 10000000.00,
            "sellingPrice": 8500000.00
          }
        ]
      },
      {
        "tripId": "550e8400-e29b-41d4-a716-446655440032",
        "startDate": "2026-04-24",
        "endDate": "2026-05-04",
        "minQuota": 1,
        "maxQuota": 25,
        "availableSeats": 14,
        "status": "ACTIVE",
        "hasTripItineraryOverride": false,
        "pricings": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440051",
            "ageBand": "ADULT",
            "ageMin": 12,
            "ageMax": null,
            "consumesQuota": true,
            "basePrice": 32000000.00,
            "sellingPrice": 28000000.00
          }
        ]
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

---

## 3. TypeScript DTOs & Interfaces

### 3.1 Variant Catalog Card DTO (`VariantCardDto`)

```typescript
export interface VariantBadgeDto {
  id: string;
  code: string;
  label: string;
  backgroundColor: string;
  textColor: string;
  iconUrl?: string | null;
}

export interface VariantCardCategoryDto {
  id: string;
  name: string;
  slug: string;
}

export interface VariantCardDestinationDto {
  poi: string;
  country: string;
  subContinent: string;
  continent: string;
}

export interface VariantCardDto {
  variantId: string;
  code: string;
  name: string;
  slug: string;
  variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';
  badges: VariantBadgeDto[];
  productId: string;
  productName: string;
  productSlug: string;
  category: VariantCardCategoryDto;
  parentCategory: VariantCardCategoryDto;
  durationDays: number;
  durationNights: number;
  coverUrl: string;
  destinations: VariantCardDestinationDto[];
  startingPrice: number;
  currency: string;
  nextDepartureDate: string | null;
  totalActiveDepartures: number;
}
```
