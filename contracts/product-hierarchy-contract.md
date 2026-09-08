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
import { IsOptional, IsString, IsInt, Min, IsIn, IsArray } from 'class-validator';
import { Type } from 'class-transformer';

export class ListVariantsDto {
  @IsOptional()
  @IsString()
  @IsIn(['STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL'])
  variantType?: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  categorySlugs?: string[];

  @IsOptional()
  @IsString()
  categorySlug?: string;

  @IsOptional()
  @IsString()
  travelStyleSlug?: string;

  @IsOptional()
  @IsString()
  themeSlug?: string;

  @IsOptional()
  @IsString()
  seasonSlug?: string;

  @IsOptional()
  @IsString()
  specialSlug?: string;

  @IsOptional()
  @IsString()
  badgeCode?: string;

  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  badgeCodes?: string[];

  @IsOptional()
  @IsString()
  @IsIn(['SHORT', 'MEDIUM', 'LONG'])
  durationBracket?: 'SHORT' | 'MEDIUM' | 'LONG';

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
      "categories": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440081",
          "name": "Popular Group Tours",
          "slug": "popular-group-tours",
          "dimensionCode": "TRAVEL_STYLE",
          "isPrimary": true
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440085",
          "name": "Spring & Sakura Season",
          "slug": "spring-sakura-season",
          "dimensionCode": "SEASON_MOMENT",
          "isPrimary": true
        }
      ],
      "durationDays": 7,
      "durationNights": 6,
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "destinations": [
        {
          "poi": "Keukenhof",
          "country": "Netherlands",
          "countryCode": "NL",
          "subContinent": "Western Europe",
          "continent": "Europe",
          "poiSlug": "keukenhof",
          "countrySlug": "netherlands",
          "subContinentSlug": "western-europe",
          "continentSlug": "europe"
        },
        {
          "poi": "Eiffel Tower",
          "country": "France",
          "countryCode": "FR",
          "subContinent": "Western Europe",
          "continent": "Europe",
          "poiSlug": "eiffel-tower",
          "countrySlug": "france",
          "subContinentSlug": "western-europe",
          "continentSlug": "europe"
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
      "categories": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440084",
          "name": "Sakura & Flower Blooms",
          "slug": "sakura-flower-blooms",
          "dimensionCode": "THEME_INTEREST",
          "isPrimary": true
        }
      ],
      "durationDays": 7,
      "durationNights": 6,
      "coverUrl": "https://cdn.hobiholidays.com/products/gwe/keukenhof-tulips.jpg",
      "destinations": [
        {
          "poi": "Keukenhof",
          "country": "Netherlands",
          "subContinent": "Western Europe",
          "continent": "Europe",
          "poiSlug": "keukenhof",
          "countrySlug": "netherlands",
          "subContinentSlug": "western-europe",
          "continentSlug": "europe"
        }
      ],
      "startingPrice": 31000000.00,
      "currency": "IDR",
      "nextDepartureDate": "2026-05-02",
      "totalActiveDepartures": 2
    }
  ]
}
      "destinations": [
        {
          "continent": "Asia",
          "subContinent": "East Asia",
          "country": "Japan",
          "poi": null
        }
      ],
      "startingPrice": 22500000.00,
      "currency": "IDR",
      "nextDepartureDate": "2026-10-15",
      "totalActiveDepartures": 3
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
      "durationDays": 7,
      "durationNights": 6,
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
          "name": "Heritage & Culture",
          "slug": "heritage-culture"
        },
        {
          "dimension": "SEASON_MOMENT",
          "dimensionName": "Season & Moment",
          "id": "550e8400-e29b-41d4-a716-446655440083",
          "name": "Spring Cherry Blossom",
          "slug": "spring-blossom"
        },
        {
          "dimension": "SPECIAL_EXPERIENCE",
          "dimensionName": "Special Experience",
          "id": "550e8400-e29b-41d4-a716-446655440084",
          "name": "Scenic Train Rides",
          "slug": "scenic-trains"
        }
      ],
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
      "title": "7D/6N Spring Blossom Western Europe",
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
        "endDate": "2026-04-16",
        "minQuota": 1,
        "maxQuota": 25,
        "availableSeats": 8,
        "status": "ACTIVE",
        "hasTripItineraryOverride": false,
        "pricings": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440041",
            "ageBand": "ADULT",
            "minAge": 12,
            "maxAge": null,
            "consumesQuota": true,
            "basePrice": 32000000.00,
            "sellingPrice": 28000000.00,
            "currency": "IDR",
            "components": [
              {
                "componentType": "BASE_FARE",
                "name": "Base Departure & Land Tour",
                "amount": 26000000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "VISA",
                "name": "Schengen Visa Fee",
                "amount": 1500000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "AIRPORT_TAX",
                "name": "Airport Tax & Fuel Surcharge",
                "amount": 350000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "TIPPING",
                "name": "Tour Leader & Driver Tipping",
                "amount": 150000.00,
                "currency": "IDR",
                "isIncluded": true
              }
            ]
          },
          {
            "id": "550e8400-e29b-41d4-a716-446655440044",
            "ageBand": "INFANT",
            "minAge": 0,
            "maxAge": 2,
            "consumesQuota": false,
            "basePrice": 10000000.00,
            "sellingPrice": 8500000.00,
            "currency": "IDR",
            "components": [
              {
                "componentType": "BASE_FARE",
                "name": "Base Infant Transport & Handling",
                "amount": 8500000.00,
                "currency": "IDR",
                "isIncluded": true
              }
            ]
          }
        ]
      },
      {
        "tripId": "550e8400-e29b-41d4-a716-446655440032",
        "startDate": "2026-04-24",
        "endDate": "2026-04-30",
        "minQuota": 1,
        "maxQuota": 25,
        "availableSeats": 14,
        "status": "ACTIVE",
        "hasTripItineraryOverride": false,
        "pricings": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440051",
            "ageBand": "ADULT",
            "minAge": 12,
            "maxAge": null,
            "consumesQuota": true,
            "basePrice": 32000000.00,
            "sellingPrice": 28000000.00,
            "currency": "IDR",
            "components": [
              {
                "componentType": "BASE_FARE",
                "name": "Base Departure & Land Tour",
                "amount": 26000000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "VISA",
                "name": "Schengen Visa Fee",
                "amount": 1500000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "AIRPORT_TAX",
                "name": "Airport Tax & Fuel Surcharge",
                "amount": 350000.00,
                "currency": "IDR",
                "isIncluded": true
              },
              {
                "componentType": "TIPPING",
                "name": "Tour Leader & Driver Tipping",
                "amount": 150000.00,
                "currency": "IDR",
                "isIncluded": true
              }
            ]
          }
        ]
      }
    ],
    "seo": {
      "metaTitle": "Tour GWE Spring 2026 (7D/6N) Eropa Barat Murah | Hobiholidays",
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
  dimensionCode: 'TRAVEL_STYLE' | 'THEME_INTEREST' | 'SEASON_MOMENT' | 'SPECIAL_EXPERIENCE';
  dimensionName: string;
}

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

export type VariantCardDestinationDto = DestinationHierarchyDto;

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
  categories: VariantCardCategoryDto[];
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
