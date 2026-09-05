# Product Domain API Contracts

> **Overview**
> Complete REST API contract specifications for the Product Domain. In accordance with micro-frontend and clean REST standards, **Product retrieval is split into dedicated, granular sub-resource endpoints** (`/media`, `/locations`, `/variants`, `/supplementaries`, `/seo`) to eliminate payload bloat, support tabbed UI loading, and maximize edge cacheability.
>
> **Core Architectural Principles:**
> - **Category Taxonomy:** 2-tier parent-child category tree (`product_categories`) linked to Products.
> - **Itinerary Hierarchy:** Owned at **Variant level (L2)** as default master itinerary, with optional override at **Trip level (L3)**.
> - **Pricing & Add-on Architecture:** Base price is all-inclusive, scoped by age band (`ADULT`, `INFANT`) with dynamic `consumes_quota` boolean flag (infants may consume quota if seat is allocated). Excluded optional extras are modeled via `product_addons`.
> - **Add-on Subsystem:** Full-price package base model with optional add-ons (`product_addons`) linked to Variants and optional Trip overrides.
> - **Itinerary PDF Brochure:** Generated externally by ATW. Endpoints store and return the CDN/ATW URL (`itineraryPdfUrl`), with Variant override falling back to Base Product.
>
> **Related Design Document:** [Product Technical Design](../technical/product-technical-design.md)  
> **Backend Guide:** [Product Backend Guide](../backend/product-backend-guide.md)  
> **Frontend Guide:** [Product Frontend Guide](../frontend/product-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Product Categories** | `GET` | `/api/v1/categories/tree` | Global 2-tier parent-child category tree |
| | `GET` | `/api/v1/categories` | List all product categories |
| | `POST` | `/api/v1/categories` | Create product category (parent or child) |
| **Base Product (L1)** | `POST` | `/api/v1/products` | Create new master product + journey duration + category binding |
| | `GET` | `/api/v1/products` | List all master products with pagination, category filter & status |
| | `GET` | `/api/v1/products/:id` | **Base Product Details** (headline, category, duration, brochure URL) |
| | `PUT` | `/api/v1/products/:id` | Update master product base info & category |
| | `DELETE`| `/api/v1/products/:id` | Soft delete master product |
| **Split Sub-Resources** | `GET` | `/api/v1/products/:id/media` | **Product Media** (covers, galleries, brochure) |
| | `GET` | `/api/v1/products/:id/locations` | **Product Locations** (destination markers & 4-tier Area tree) |
| | `GET` | `/api/v1/products/:id/variants` | **Product Variants** (L2 packages under this master product) |
| | `GET` | `/api/v1/products/:id/supplementaries`| **Product Supplementary** (inclusions, exclusions, terms) |
| | `GET` | `/api/v1/products/:id/seo` | **Product SEO Metadata** (custom meta title, description, OG) |
| | `PUT` | `/api/v1/products/:id/seo` | Upsert product custom SEO metadata |
| **Sub-Resource Mutations**| `POST` | `/api/v1/products/:id/locations` | Attach destination marker (4-tier Area) |
| | `POST` | `/api/v1/products/:id/supplementaries`| Add supplementary block |
| **L2 Variants** | `POST` | `/api/v1/products/:id/variants` | Create variant (Standard, Seasonal, Themed, etc.) |
| | `GET` | `/api/v1/variants/:id` | Fetch specific variant details |
| | `PUT` | `/api/v1/variants/:id` | Update variant title, slug, duration override |
| **L2 Variant Itinerary** | `GET` | `/api/v1/variants/:variantId/itinerary` | Fetch default master itinerary for variant |
| | `PUT` | `/api/v1/variants/:variantId/itinerary` | Upsert/replace default master itinerary for variant |
| **L2 Variant Add-ons** | `GET` | `/api/v1/variants/:variantId/addons` | List optional add-ons configured for variant |
| | `POST` | `/api/v1/variants/:variantId/addons` | Create optional add-on for variant |
| | `PUT` | `/api/v1/addons/:id` | Update add-on details |
| | `DELETE`| `/api/v1/addons/:id` | Soft delete add-on |
| **L3 Trips** | `GET` | `/api/v1/variants/:variantId/trips`| List trips (dated departures & quotas) |
| | `POST` | `/api/v1/variants/:variantId/trips`| Create dated departure window |
| **L3 Trip Itinerary** | `GET` | `/api/v1/trips/:tripId/itinerary` | Fetch trip-specific itinerary override (if any) |
| | `PUT` | `/api/v1/trips/:tripId/itinerary` | Upsert trip-specific itinerary override |
| | `DELETE`| `/api/v1/trips/:tripId/itinerary` | Remove override (reverts to variant default) |
| | `GET` | `/api/v1/trips/:tripId/effective-itinerary`| Resolved itinerary (`trip ?? variant`) with `isOverride` flag |
| **L3 Trip Pricing** | `GET` | `/api/v1/trips/:tripId/pricings` | List pricing tiers by age band with itemized components |
| | `PUT` | `/api/v1/trips/:tripId/pricings` | Upsert pricing tier with breakdown components |
| **Promotional Badges** | `GET` | `/api/v1/badges` | List all active promotional badges |
| | `POST` | `/api/v1/badges` | Create promotional badge (Admin) |
| | `PUT` | `/api/v1/badges/:id` | Update promotional badge (Admin) |
| | `DELETE`| `/api/v1/badges/:id` | Soft delete/deactivate promotional badge (Admin) |
| | `POST` | `/api/v1/variants/:id/badges` | Attach promotional badges to variant |
| | `DELETE`| `/api/v1/variants/:id/badges/:badgeId` | Detach promotional badge from variant |

---

## 1. Product Category Endpoints

### 1.1 Category Hierarchy Tree (`GET /api/v1/categories/tree`)
Returns the complete 2-tier parent-child category tree. Cached with 24-hour TTL for storefront navigation and catalog filtering.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Category tree retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440080",
      "name": "Tour Series",
      "slug": "tour-series",
      "description": "Standard scheduled group departure series",
      "sortOrder": 1,
      "children": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440081",
          "name": "Classic Series",
          "slug": "classic-series",
          "description": "Flagship classic itineraries covering iconic highlights",
          "sortOrder": 1
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440082",
          "name": "Flower Season",
          "slug": "flower-season",
          "description": "Seasonal bloom and festival focused journeys",
          "sortOrder": 2
        }
      ]
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440090",
      "name": "Special Interest",
      "slug": "special-interest",
      "description": "Themed and interest-based experiential travels",
      "sortOrder": 2,
      "children": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440091",
          "name": "Culinary & Wine",
          "slug": "culinary-wine",
          "description": "Gastronomic experiences and vineyard visits",
          "sortOrder": 1
        }
      ]
    }
  ]
}
```

---

## 2. Base Product Endpoints

### 2.1 List Products (`GET /api/v1/products`)
Returns a paginated list of master products, filterable by status, product type, and category.

#### Query Parameters (`ListProductsDto`)
```typescript
import { IsOptional, IsString, IsIn, IsInt, Min, IsUUID } from 'class-validator';
import { Type } from 'class-transformer';

export class ListProductsDto {
  @IsOptional()
  @IsString()
  @IsIn(['DRAFT', 'PENDING_REVIEW', 'ACTIVE', 'INACTIVE', 'ARCHIVED', 'SUSPENDED'])
  listingStatus?: string;

  @IsOptional()
  @IsString()
  @IsIn(['JOURNEY', 'OPEN_TRIP', 'PRIVATE_TRIP', 'DAY_TOUR'])
  productType?: string;

  @IsOptional()
  @IsUUID('4')
  categoryId?: string;

  @IsOptional()
  @IsString()
  categorySlug?: string;

  @IsOptional()
  @IsString()
  search?: string; // Partial name / code search

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

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Products retrieved successfully",
  "meta": {
    "totalItems": 24,
    "itemCount": 2,
    "itemsPerPage": 10,
    "totalPages": 3,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "code": "GWE",
      "name": "Grand West Europe",
      "slug": "grand-west-europe",
      "productType": "JOURNEY",
      "listingStatus": "ACTIVE",
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
      "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf",
      "variantsCount": 5,
      "createdAt": "2026-09-04T08:00:00.000Z",
      "updatedAt": "2026-09-04T09:30:00.000Z"
    }
  ]
}
```

---

### 2.2 Create Product (`POST /api/v1/products`)
Creates a new master tour product with category binding and journey duration.

#### Request DTO (`CreateProductDto`)
```typescript
import { IsString, IsIn, IsInt, Min, IsUUID } from 'class-validator';

export class CreateProductDto {
  @IsString()
  code: string; // e.g. "GWE-MASTER"

  @IsString()
  name: string; // e.g. "Grand West Europe"

  @IsString()
  slug: string; // e.g. "grand-west-europe"

  @IsUUID('4')
  categoryId: string; // Child category UUID (parent_category_id auto-resolved by trigger)

  @IsString()
  @IsIn(['JOURNEY', 'OPEN_TRIP', 'PRIVATE_TRIP', 'DAY_TOUR'])
  productType: 'JOURNEY' | 'OPEN_TRIP' | 'PRIVATE_TRIP' | 'DAY_TOUR';

  @IsInt()
  @Min(1)
  durationDays: number; // e.g. 11

  @IsInt()
  @Min(0)
  durationNights: number; // e.g. 10
}
```

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Product created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "code": "GWE-MASTER",
    "name": "Grand West Europe",
    "slug": "grand-west-europe",
    "categoryId": "550e8400-e29b-41d4-a716-446655440081",
    "parentCategoryId": "550e8400-e29b-41d4-a716-446655440080",
    "productType": "JOURNEY",
    "listingStatus": "DRAFT",
    "durationDays": 11,
    "durationNights": 10,
    "itineraryPdfUrl": "https://atw-cdn.hobiholidays.com/brochures/gwe-master-brochure.pdf",
    "createdAt": "2026-09-04T10:00:00.000Z"
  }
}
```

---

## 3. Split Sub-Resource Endpoints

### 3.1 Get Product Locations (`GET /api/v1/products/:id/locations`)
Returns destination markers linked to the 4-tier Area domain (**Continent → Sub Continent → Country → POI**).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product locations retrieved successfully",
  "data": [
    {
      "locationId": "550e8400-e29b-41d4-a716-446655440081",
      "areaId": "550e8400-e29b-41d4-a716-446655440010",
      "poi": "Keukenhof",
      "country": "Netherlands",
      "countryCode": "NL",
      "subContinent": "Western Europe",
      "continent": "Europe",
      "lat": 52.2698,
      "lng": 4.5469,
      "sortOrder": 1
    }
  ]
}
```

---

## 4. L2 Variant Endpoints & Master Itinerary

### 4.1 Create Variant (`POST /api/v1/products/:id/variants`)
```typescript
import { IsString, IsIn, IsInt, Min, IsOptional } from 'class-validator';

export class CreateVariantDto {
  @IsString()
  code: string; // e.g. "GWE-SPR-2026"

  @IsString()
  name: string; // e.g. "GWE Spring 2026"

  @IsString()
  slug: string; // e.g. "gwe-spring-2026"

  @IsString()
  @IsIn(['STANDARD', 'SEASONAL', 'THEMED', 'PROMOTIONAL'])
  variantType: 'STANDARD' | 'SEASONAL' | 'THEMED' | 'PROMOTIONAL';

  @IsOptional()
  @IsInt()
  @Min(1)
  durationDays?: number;

  @IsOptional()
  @IsInt()
  @Min(0)
  durationNights?: number;

  @IsOptional()
  @IsString()
  itineraryPdfUrl?: string; // Optional variant-specific ATW brochure URL override
}
```

---

### 4.2 Get Variant Master Itinerary (`GET /api/v1/variants/:variantId/itinerary`)
Returns the default master day-by-day itinerary defined for this variant.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Variant default itinerary retrieved successfully",
  "data": {
    "itineraryId": "550e8400-e29b-41d4-a716-446655440070",
    "variantId": "550e8400-e29b-41d4-a716-446655440020",
    "tripId": null,
    "title": "GWE Spring Master Itinerary 11D",
    "daysCount": 11,
    "items": [
      {
        "itemId": "550e8400-e29b-41d4-a716-446655440071",
        "dayNumber": 1,
        "sequenceNumber": 1,
        "itemType": "TRANSPORT",
        "title": "Jakarta - Amsterdam",
        "description": "Boarding direct flight to Amsterdam.",
        "meals": { "breakfast": false, "lunch": false, "dinner": true },
        "accommodation": "In-flight",
        "locationName": "Amsterdam"
      },
      {
        "itemId": "550e8400-e29b-41d4-a716-446655440072",
        "dayNumber": 2,
        "sequenceNumber": 1,
        "itemType": "ACTIVITY",
        "title": "Amsterdam - Keukenhof",
        "description": "Visit Keukenhof tulip gardens and Zaanse Schans.",
        "meals": { "breakfast": true, "lunch": true, "dinner": true },
        "accommodation": "Van der Valk Hotel Amsterdam",
        "locationName": "Keukenhof"
      },
      {
        "itemId": "550e8400-e29b-41d4-a716-446655440073",
        "dayNumber": 3,
        "sequenceNumber": 1,
        "itemType": "OTHER",
        "title": "Free Leisure & Personal Exploration",
        "description": "Acclimatization, shopping, or personal exploration around Amsterdam city center.",
        "meals": { "breakfast": true, "lunch": false, "dinner": false },
        "accommodation": "Van der Valk Hotel Amsterdam",
        "locationName": "Amsterdam"
      }
    ]
  }
}
```

---

### 4.3 Upsert Variant Master Itinerary (`PUT /api/v1/variants/:variantId/itinerary`)

#### Request DTO (`UpsertItineraryDto`)
```typescript
import { IsString, IsArray, ValidateNested, IsInt, Min, IsOptional, IsBoolean, IsIn, IsUUID } from 'class-validator';
import { Type } from 'class-transformer';

export class ItineraryDayMealDto {
  @IsBoolean() breakfast: boolean;
  @IsBoolean() lunch: boolean;
  @IsBoolean() dinner: boolean;
}

export class ItineraryItemDto {
  @IsInt()
  @Min(1)
  dayNumber: number;

  @IsInt()
  @Min(1)
  sequenceNumber: number;

  @IsIn(['ACTIVITY', 'TRANSPORT', 'MEAL', 'ACCOMMODATION', 'OTHER'])
  itemType: 'ACTIVITY' | 'TRANSPORT' | 'MEAL' | 'ACCOMMODATION' | 'OTHER';

  @IsString()
  title: string;

  @IsString()
  description: string;

  @IsOptional()
  @ValidateNested()
  @Type(() => ItineraryDayMealDto)
  meals?: ItineraryDayMealDto;

  @IsOptional()
  @IsString()
  accommodation?: string;

  @IsOptional()
  @IsString()
  locationName?: string;

  @IsOptional()
  @IsUUID()
  poiAreaId?: string;
}

export class UpsertItineraryDto {
  @IsString()
  title: string;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ItineraryItemDto)
  items: ItineraryItemDto[];
}
```

---

## 5. L2 Variant Add-on Subsystem

### 5.1 List Variant Add-ons (`GET /api/v1/variants/:variantId/addons`)
Returns all optional or required add-ons available for selection on this variant.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Variant add-ons retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440070",
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "code": "ADDON-VISA-FAST",
      "name": "Schengen Visa Fast Track",
      "description": "Priority appointment and consular processing assistance",
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
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "code": "ADDON-EIFFEL-SUMMIT",
      "name": "Eiffel Tower Summit Access",
      "description": "Skip-the-line elevator ticket to top observation deck",
      "addonType": "EXPERIENTIAL_TOUR",
      "chargeType": "PER_PAX",
      "price": 750000.00,
      "currency": "IDR",
      "applicableAgeBand": "ADULT",
      "isMandatory": false,
      "maxQuantity": 4
    }
  ]
}
```

---

### 5.2 Create Variant Add-on (`POST /api/v1/variants/:variantId/addons`)

#### Request DTO (`CreateAddonDto`)
```typescript
import { IsString, IsNumber, IsPositive, IsIn, IsBoolean, IsOptional, IsInt, Min } from 'class-validator';

export class CreateAddonDto {
  @IsString()
  code: string;

  @IsString()
  name: string;

  @IsOptional()
  @IsString()
  description?: string;

  @IsString()
  @IsIn(['SINGLE_ROOM', 'BAGGAGE', 'FLIGHT_UPGRADE', 'EXPERIENTIAL_TOUR', 'INSURANCE', 'VISA_EXPRESS', 'SPECIAL_MEAL'])
  addonType: string;

  @IsString()
  @IsIn(['PER_PAX', 'PER_ROOM', 'PER_BOOKING'])
  chargeType: 'PER_PAX' | 'PER_ROOM' | 'PER_BOOKING';

  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  price: number;

  @IsOptional()
  @IsString()
  currency?: string = 'IDR';

  @IsOptional()
  @IsString()
  @IsIn(['ADULT', 'INFANT'])
  applicableAgeBand?: 'ADULT' | 'INFANT'; // Optional age-band restriction; null/omitted = applies to ALL

  @IsOptional()
  @IsBoolean()
  isMandatory?: boolean = false;

  @IsOptional()
  @IsInt()
  @Min(1)
  maxQuantity?: number = 1;
}
```

---

## 6. L3 Trip Itinerary & Effective Resolution

### 6.1 Get Effective Itinerary (`GET /api/v1/trips/:tripId/effective-itinerary`)
Resolves itinerary following the priority rule: returns **Trip-specific override** if present; otherwise falls back to **Variant default master itinerary**.

#### Success Response (200 OK - Fallback to Variant Default)
```json
{
  "statusCode": 200,
  "message": "Effective itinerary resolved",
  "data": {
    "itineraryId": "550e8400-e29b-41d4-a716-446655440070",
    "tripId": "550e8400-e29b-41d4-a716-446655440031",
    "variantId": "550e8400-e29b-41d4-a716-446655440020",
    "isOverride": false,
    "title": "GWE Spring Master Itinerary 11D",
    "daysCount": 11,
    "items": [
      {
        "dayNumber": 1,
        "sequenceNumber": 1,
        "itemType": "TRANSPORT",
        "title": "Jakarta - Amsterdam",
        "description": "Boarding direct flight to Amsterdam."
      }
    ]
  }
}
```

#### Success Response (200 OK - Trip Override Active)
```json
{
  "statusCode": 200,
  "message": "Effective itinerary resolved",
  "data": {
    "itineraryId": "550e8400-e29b-41d4-a716-446655440079",
    "tripId": "550e8400-e29b-41d4-a716-446655440031",
    "variantId": "550e8400-e29b-41d4-a716-446655440020",
    "isOverride": true,
    "title": "GWE Spring 11D - Keukenhof Peak Special (Trip Override)",
    "daysCount": 11,
    "items": [
      {
        "dayNumber": 1,
        "sequenceNumber": 1,
        "itemType": "TRANSPORT",
        "title": "Jakarta - Doha - Amsterdam",
        "description": "Flight via Doha with private lounge layover."
      }
    ]
  }
}
```

---

## 7. L3 Trip Pricing & Age Bands

### 7.1 List Trip Pricings (`GET /api/v1/trips/:tripId/pricings`)
Returns pricing tiers for each traveler age band (`ADULT`, `INFANT`) along with whether each consumes seat quota (`consumesQuota`).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Trip pricings retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440041",
      "tripId": "550e8400-e29b-41d4-a716-446655440031",
      "ageBand": "ADULT",
      "ageMin": 12,
      "ageMax": null,
      "consumesQuota": true,
      "basePrice": 32000000.00,
      "sellingPrice": 28000000.00,
      "currency": "IDR",
      "components": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440051",
          "name": "International Flight & Taxes",
          "amount": 14000000.00,
          "isIncluded": true,
          "description": "Economy return flight with Qatar Airways"
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440052",
          "name": "4-Star Hotel Accommodation (Twin)",
          "amount": 8000000.00,
          "isIncluded": true,
          "description": "9 nights twin-sharing in 4-star hotels"
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440053",
          "name": "Private Coach & Transfers",
          "amount": 3500000.00,
          "isIncluded": true,
          "description": "Air-conditioned private touring motorcoach"
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440054",
          "name": "Tour Leader & Local Guides",
          "amount": 1500000.00,
          "isIncluded": true,
          "description": "Professional Indonesian tour leader"
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440055",
          "name": "Keukenhof & Attraction Admissions",
          "amount": 1000000.00,
          "isIncluded": true,
          "description": "Fast-track entrance tickets"
        }
      ]
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440044",
      "tripId": "550e8400-e29b-41d4-a716-446655440031",
      "ageBand": "INFANT",
      "ageMin": 0,
      "ageMax": 2,
      "consumesQuota": false,
      "basePrice": 10000000.00,
      "sellingPrice": 8500000.00,
      "currency": "IDR",
      "components": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440058",
          "name": "Infant Airline Ticket & Tax",
          "amount": 7500000.00,
          "isIncluded": true,
          "description": "Lap infant international ticket"
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440059",
          "name": "Infant Travel Insurance & Admin",
          "amount": 1000000.00,
          "isIncluded": true,
          "description": "Comprehensive infant travel insurance"
        }
      ]
    }
  ]
}
```

---

### 7.2 Upsert Trip Pricing with Breakdown Components (`PUT /api/v1/trips/:tripId/pricings`)

#### Request DTO (`UpsertTripPricingDto`)
```typescript
import {
  IsString,
  IsIn,
  IsNumber,
  IsPositive,
  IsBoolean,
  IsOptional,
  IsInt,
  Min,
  ValidateNested,
  IsArray,
} from 'class-validator';
import { Type } from 'class-transformer';

export class CreatePricingComponentDto {
  @IsString()
  name: string; // e.g. "International Flight & Taxes", "4-Star Hotel Accommodation"

  @IsOptional()
  @IsString()
  description?: string;

  @IsOptional()
  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  amount?: number;

  @IsOptional()
  @IsBoolean()
  isIncluded?: boolean = true;

  @IsOptional()
  @IsInt()
  @Min(0)
  sortOrder?: number = 0;
}

export class UpsertTripPricingDto {
  @IsString()
  @IsIn(['ADULT', 'INFANT'])
  ageBand: 'ADULT' | 'INFANT';

  @IsOptional()
  @IsInt()
  @Min(0)
  ageMin?: number;

  @IsOptional()
  @IsInt()
  @Min(0)
  ageMax?: number;

  @IsOptional()
  @IsBoolean()
  consumesQuota?: boolean = true; // Configurable: true if seat allocated, false if lap infant

  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  basePrice: number;

  @IsNumber({ maxDecimalPlaces: 2 })
  @IsPositive()
  sellingPrice: number;

  @IsOptional()
  @IsString()
  currency?: string = 'IDR';

  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => CreatePricingComponentDto)
  components?: CreatePricingComponentDto[];
}
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Trip pricing upserted successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440041",
    "tripId": "550e8400-e29b-41d4-a716-446655440031",
    "ageBand": "ADULT",
    "consumesQuota": true,
    "basePrice": 32000000.00,
    "sellingPrice": 28000000.00,
    "components": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440051",
        "name": "International Flight & Taxes",
        "amount": 14000000.00,
        "isIncluded": true,
        "description": "Economy return flight with Qatar Airways"
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440052",
        "name": "4-Star Hotel Accommodation (Twin)",
        "amount": 8000000.00,
        "isIncluded": true,
        "description": "9 nights twin-sharing in 4-star hotels"
      }
    ]
  }
}
```

---

## 8. Promotional Badges Endpoints

### 8.1 List All Badges (`GET /api/v1/badges`)
Returns all active promotional badges configured for the platform.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Badges retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440090",
      "code": "BEST_SELLER",
      "label": "🔥 Best Seller",
      "backgroundColor": "#FEF2F2",
      "textColor": "#991B1B",
      "iconUrl": null,
      "isActive": true
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440091",
      "code": "SPRING_EDITION",
      "label": "🌸 Spring Edition",
      "backgroundColor": "#FDF2F8",
      "textColor": "#9D174D",
      "iconUrl": null,
      "isActive": true
    }
  ]
}
```

### 8.2 Create Badge (`POST /api/v1/badges`)

#### Request DTO (`CreateBadgeDto`)
```typescript
import { IsString, IsNotEmpty, IsOptional, IsBoolean } from 'class-validator';

export class CreateBadgeDto {
  @IsString()
  @IsNotEmpty()
  code: string; // e.g. 'BEST_SELLER', 'SPRING_EDITION'

  @IsString()
  @IsNotEmpty()
  label: string; // e.g. '🔥 Best Seller', '🌸 Spring Edition'

  @IsOptional()
  @IsString()
  backgroundColor?: string = '#F3F4F6';

  @IsOptional()
  @IsString()
  textColor?: string = '#1F2937';

  @IsOptional()
  @IsString()
  iconUrl?: string;

  @IsOptional()
  @IsBoolean()
  isActive?: boolean = true;
}
```

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Badge created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440092",
    "code": "EARLY_BIRD",
    "label": "⚡ Early Bird",
    "backgroundColor": "#FEFCE8",
    "textColor": "#854D0E",
    "iconUrl": null,
    "isActive": true
  }
}
```

### 8.3 Assign Badges to Variant (`POST /api/v1/variants/:id/badges`)

#### Request DTO (`AssignVariantBadgesDto`)
```typescript
import { IsArray, IsUUID } from 'class-validator';

export class AssignVariantBadgesDto {
  @IsArray()
  @IsUUID('4', { each: true })
  badgeIds: string[];
}
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Badges assigned to variant successfully",
  "data": {
    "variantId": "550e8400-e29b-41d4-a716-446655440020",
    "badges": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440090",
        "code": "BEST_SELLER",
        "label": "🔥 Best Seller",
        "backgroundColor": "#FEF2F2",
        "textColor": "#991B1B"
      },
      {
        "id": "550e8400-e29b-41d4-a716-446655440091",
        "code": "SPRING_EDITION",
        "label": "🌸 Spring Edition",
        "backgroundColor": "#FDF2F8",
        "textColor": "#9D174D"
      }
    ]
  }
}
```

### 8.4 Detach Badge from Variant (`DELETE /api/v1/variants/:id/badges/:badgeId`)

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Badge detached from variant successfully",
  "data": {
    "variantId": "550e8400-e29b-41d4-a716-446655440020",
    "detachedBadgeId": "550e8400-e29b-41d4-a716-446655440090"
  }
}
```
