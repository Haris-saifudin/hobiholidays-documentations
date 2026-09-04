# Product Domain API Contracts

> **Overview**
> Complete REST API contract specifications for the Product Domain. In accordance with micro-frontend and clean REST standards, **Product retrieval is split into dedicated, granular sub-resource endpoints** (`/media`, `/itineraries`, `/locations`, `/variants`, `/supplementaries`, `/seo`) to eliminate payload bloat, support tabbed UI loading, and maximize edge cacheability.
>
> **Related Design Document:** [Product Technical Design](../product-technical-design.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Base Product** | `POST` | `/api/v1/products` | Create new master product + journey duration |
| | `GET` | `/api/v1/products` | List all master products with pagination & status |
| | `GET` | `/api/v1/products/:id` | **Base Product Details** (headline, duration, brochure URL) |
| | `PUT` | `/api/v1/products/:id` | Update master product base info |
| | `DELETE`| `/api/v1/products/:id` | Soft delete master product |
| **Split Sub-Resources** | `GET` | `/api/v1/products/:id/media` | **Product Media** (covers, galleries, brochure) |
| | `GET` | `/api/v1/products/:id/itineraries`| **Product Itinerary** (day-by-day stops & activities) |
| | `GET` | `/api/v1/products/:id/locations` | **Product Locations** (destination markers & Area tree) |
| | `GET` | `/api/v1/products/:id/variants` | **Product Variants** (L2 packages under this master product) |
| | `GET` | `/api/v1/products/:id/supplementaries`| **Product Supplementary** (inclusions, exclusions, terms) |
| | `GET` | `/api/v1/products/:id/seo` | **Product SEO Metadata** (custom meta title, description, OG) |
| | `PUT` | `/api/v1/products/:id/seo` | Upsert product custom SEO metadata |
| **Sub-Resource Mutations**| `POST` | `/api/v1/products/:id/itineraries` | Create/replace master itinerary |
| | `POST` | `/api/v1/products/:id/locations` | Attach destination marker |
| | `POST` | `/api/v1/products/:id/supplementaries`| Add supplementary block |
| **L2 Variants** | `POST` | `/api/v1/products/:id/variants` | Create variant (Standard, Seasonal, Themed, etc.) |
| | `GET` | `/api/v1/variants/:id` | Fetch specific variant details |
| | `PUT` | `/api/v1/variants/:id` | Update variant title, slug, duration override |
| **L3 Trips & Pricing** | `GET` | `/api/v1/variants/:variantId/trips`| List trips (dated departures & quotas) |
| | `POST` | `/api/v1/variants/:variantId/trips`| Create dated departure window |
| | `GET` | `/api/v1/trips/:tripId/pricings` | List pricing tiers (ALL, DOMESTIC, INTERNATIONAL) |
| | `PUT` | `/api/v1/trips/:tripId/pricings` | Upsert pricing tier |

---

## 1. Base Product Endpoints

### 1.1 List Products (`GET /api/v1/products`)
Returns a paginated list of master products, filterable by listing status and product type.

#### Query Parameters (`ListProductsDto`)

```typescript
import { IsOptional, IsString, IsIn, IsInt, Min } from 'class-validator';
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
      "durationDays": 7,
      "durationNights": 6,
      "nationalityScope": "ALL",
      "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf",
      "variantsCount": 5,
      "createdAt": "2026-09-04T08:00:00.000Z",
      "updatedAt": "2026-09-04T09:30:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440011",
      "code": "TURKEY-WONDERS",
      "name": "Turkey Wonders",
      "slug": "turkey-wonders",
      "productType": "JOURNEY",
      "listingStatus": "ACTIVE",
      "durationDays": 9,
      "durationNights": 8,
      "nationalityScope": "ALL",
      "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/turkey-wonders.pdf",
      "variantsCount": 4,
      "createdAt": "2026-09-04T08:30:00.000Z",
      "updatedAt": "2026-09-04T10:00:00.000Z"
    }
  ]
}
```

---

### 1.2 Create Product (`POST /api/v1/products`)
Creates a new master tour product with initial base journey duration.

#### Request DTO
```typescript
import { IsString, IsIn, IsInt, Min, IsOptional } from 'class-validator';

export class CreateProductDto {
  @IsString()
  code: string; // e.g. "TURKEY-WONDERS"

  @IsString()
  name: string; // e.g. "Turkey Wonders"

  @IsString()
  slug: string; // e.g. "turkey-wonders"

  @IsString()
  @IsIn(['JOURNEY', 'OPEN_TRIP', 'PRIVATE_TRIP', 'DAY_TOUR'])
  productType: 'JOURNEY' | 'OPEN_TRIP' | 'PRIVATE_TRIP' | 'DAY_TOUR';

  @IsInt()
  @Min(1)
  durationDays: number; // e.g. 9

  @IsInt()
  @Min(0)
  durationNights: number; // e.g. 7

  @IsOptional()
  @IsString()
  @IsIn(['ALL', 'DOMESTIC', 'INTERNATIONAL'])
  nationalityScope?: 'ALL' | 'DOMESTIC' | 'INTERNATIONAL' = 'ALL';
}
```

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Product created successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "code": "TURKEY-WONDERS",
    "name": "Turkey Wonders",
    "slug": "turkey-wonders",
    "productType": "JOURNEY",
    "listingStatus": "DRAFT",
    "durationDays": 9,
    "durationNights": 7,
    "nationalityScope": "ALL",
    "itineraryPdfUrl": null,
    "createdAt": "2026-09-04T10:00:00.000Z"
  }
}
```

---

### 1.3 Get Base Product (`GET /api/v1/products/:id`)
Fetches high-level headline information and base journey metadata. Does **not** include heavy nested collections (media, full itinerary, etc.).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440010",
    "code": "GWE",
    "name": "Grand West Europe",
    "slug": "grand-west-europe",
    "productType": "JOURNEY",
    "listingStatus": "ACTIVE",
    "durationDays": 11,
    "durationNights": 9,
    "nationalityScope": "ALL",
    "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf",
    "createdAt": "2026-09-04T08:00:00.000Z",
    "updatedAt": "2026-09-04T09:30:00.000Z"
  }
}
```

---

## 2. Split Sub-Resource Endpoints

### 2.1 Get Product Media (`GET /api/v1/products/:id/media`)
Returns all media assets attached to this product, logically grouped by usage context (`cover`, `gallery`, `itineraryPdf`).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product media retrieved successfully",
  "data": {
    "productId": "550e8400-e29b-41d4-a716-446655440010",
    "cover": {
      "mediaId": "550e8400-e29b-41d4-a716-446655440050",
      "url": "https://cdn.hobiholidays.com/products/gwe/gwe-hero-paris.jpg",
      "fileName": "gwe-hero-paris.jpg",
      "fileSizeBytes": 2410500,
      "mimeType": "image/jpeg"
    },
    "gallery": [
      {
        "usageId": "550e8400-e29b-41d4-a716-446655440061",
        "mediaId": "550e8400-e29b-41d4-a716-446655440051",
        "url": "https://cdn.hobiholidays.com/products/gwe/amsterdam-canals.jpg",
        "fileName": "amsterdam-canals.jpg",
        "sortOrder": 1
      },
      {
        "usageId": "550e8400-e29b-41d4-a716-446655440062",
        "mediaId": "550e8400-e29b-41d4-a716-446655440052",
        "url": "https://cdn.hobiholidays.com/products/gwe/brussels-atomium.jpg",
        "fileName": "brussels-atomium.jpg",
        "sortOrder": 2
      }
    ],
    "itineraryPdf": {
      "mediaId": "550e8400-e29b-41d4-a716-446655440055",
      "url": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf",
      "fileName": "GWE-Official-Brochure-2026.pdf",
      "fileSizeBytes": 4613734,
      "mimeType": "application/pdf"
    }
  }
}
```

---

### 2.2 Get Product Itineraries (`GET /api/v1/products/:id/itineraries`)
Returns the active master day-by-day itinerary and chronological item timeline.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product itineraries retrieved successfully",
  "data": {
    "itineraryId": "550e8400-e29b-41d4-a716-446655440070",
    "productId": "550e8400-e29b-41d4-a716-446655440010",
    "title": "Grand West Europe Signature 11D",
    "daysCount": 11,
    "items": [
      {
        "itemId": "550e8400-e29b-41d4-a716-446655440071",
        "dayNumber": 1,
        "title": "Jakarta - Amsterdam",
        "description": "Gather at Soekarno-Hatta Airport for direct flight to Amsterdam.",
        "meals": {
          "breakfast": false,
          "lunch": false,
          "dinner": true
        },
        "accommodation": "In-flight",
        "locationName": "Amsterdam",
        "sortOrder": 0
      },
      {
        "itemId": "550e8400-e29b-41d4-a716-446655440072",
        "dayNumber": 2,
        "title": "Arrival in Amsterdam & Canal Cruise",
        "description": "Explore Zaanse Schans windmills, cheese factory, and take glass-topped canal cruise.",
        "meals": {
          "breakfast": true,
          "lunch": true,
          "dinner": true
        },
        "accommodation": "Van der Valk Hotel Amsterdam",
        "locationName": "Amsterdam",
        "sortOrder": 1
      }
    ]
  }
}
```

---

### 2.3 Get Product Locations (`GET /api/v1/products/:id/locations`)
Returns destination markers associated with the product, resolved to the 3-tier Area domain hierarchy (**Continent → Country → City**).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product locations retrieved successfully",
  "data": [
    {
      "locationId": "550e8400-e29b-41d4-a716-446655440081",
      "areaId": "550e8400-e29b-41d4-a716-446655440001",
      "city": "Amsterdam",
      "country": "Netherlands",
      "countryCode": "NL",
      "continent": "Europe",
      "lat": 52.3676,
      "lng": 4.9041,
      "address": "Centraal & Canal Ring, Amsterdam, Netherlands",
      "sortOrder": 1
    },
    {
      "locationId": "550e8400-e29b-41d4-a716-446655440082",
      "areaId": "550e8400-e29b-41d4-a716-446655440002",
      "city": "Paris",
      "country": "France",
      "countryCode": "FR",
      "continent": "Europe",
      "lat": 48.8566,
      "lng": 2.3522,
      "address": "Champs-Élysées & Eiffel, Paris, France",
      "sortOrder": 2
    }
  ]
}
```

---

### 2.4 Get Product Variants (`GET /api/v1/products/:id/variants`)
Returns all L2 variants created under this master product, along with active starting price and trip counts.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product variants retrieved successfully",
  "data": [
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "code": "GWE-SPR-2026",
      "name": "GWE Spring 2026",
      "slug": "gwe-spring-2026",
      "variantType": "SEASONAL",
      "listingStatus": "ACTIVE",
      "durationDays": 11,
      "durationNights": 9,
      "startingPrice": 28000000.00,
      "activeTripsCount": 4
    },
    {
      "variantId": "550e8400-e29b-41d4-a716-446655440021",
      "code": "GWE-TLP-2026",
      "name": "Tulip Keukenhof Special",
      "slug": "tulip-keukenhof-special",
      "variantType": "THEMED",
      "listingStatus": "ACTIVE",
      "durationDays": 9,
      "durationNights": 7,
      "startingPrice": 31000000.00,
      "activeTripsCount": 2
    }
  ]
}
```

---

### 2.5 Get Product Supplementaries (`GET /api/v1/products/:id/supplementaries`)
Returns modular supplementary content blocks (inclusions, exclusions, visa requirements, terms).

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Product supplementaries retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440091",
      "category": "INCLUDED",
      "title": "Paket Termasuk",
      "content": "<ul><li>Tiket pesawat internasional PP kelas ekonomi</li><li>Akomodasi hotel bintang 4 setaraf</li><li>Makan sesuai itinerary</li><li>Tour Leader profesional dari Jakarta</li></ul>"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440092",
      "category": "EXCLUDED",
      "title": "Paket Tidak Termasuk",
      "content": "<ul><li>Biaya pembuatan Visa Schengen</li><li>Tipping Tour Leader & Supir (€7/hari/orang)</li><li>Pengeluaran pribadi</li></ul>"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440093",
      "category": "IMPORTANT_INFO",
      "title": "Informasi Penting",
      "content": "<p>Valid passport with at least 6 months validity required from travel date.</p>"
    }
  ]
}
```

---

### 2.6 Get & Update Product SEO (`GET` & `PUT /api/v1/products/:id/seo`)

#### Get SEO (200 OK)
```json
{
  "statusCode": 200,
  "message": "SEO metadata retrieved successfully",
  "data": {
    "targetType": "PRODUCT",
    "targetId": "550e8400-e29b-41d4-a716-446655440010",
    "metaTitle": "Paket Tour Grand West Europe 11 Hari Murah | Hobiholidays",
    "metaDescription": "Nikmati paket tour Grand West Europe 11D mengunjungi Belanda, Belgia, Prancis, dan Swiss dengan fasilitas premium.",
    "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe",
    "ogTitle": "Tour Grand West Europe 11D - Keberangkatan Pasti",
    "ogDescription": "Paket liburan Eropa Barat terbaik bersama Hobiholidays.",
    "ogImageUrl": "https://cdn.hobiholidays.com/products/gwe/og-gwe.jpg",
    "noIndex": false,
    "noFollow": false
  }
}
```

#### Update SEO Request DTO (`PUT /api/v1/products/:id/seo`)
```typescript
import { IsString, IsOptional, IsBoolean, IsUrl, MaxLength } from 'class-validator';

export class UpdateProductSeoDto {
  @IsOptional()
  @IsString()
  @MaxLength(255)
  metaTitle?: string;

  @IsOptional()
  @IsString()
  @MaxLength(500)
  metaDescription?: string;

  @IsOptional()
  @IsUrl()
  canonicalUrl?: string;

  @IsOptional()
  @IsString()
  ogTitle?: string;

  @IsOptional()
  @IsString()
  ogDescription?: string;

  @IsOptional()
  @IsUrl()
  ogImageUrl?: string;

  @IsOptional()
  @IsBoolean()
  noIndex?: boolean = false;

  @IsOptional()
  @IsBoolean()
  noFollow?: boolean = false;
}
```

---

## 3. L2 Variant & L3 Trip Endpoints

### 3.1 Create Variant (`POST /api/v1/products/:id/variants`)
```typescript
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
  durationDays?: number; // Optional duration override

  @IsOptional()
  @IsInt()
  @Min(0)
  durationNights?: number;
}
```

---

### 3.2 List Trips (`GET /api/v1/variants/:variantId/trips`)
Returns a paginated list of dated departure windows under a specific variant.

#### Query Parameters
- `status` (optional): Filter by trip status (`ACTIVE`, `FULL`, `CANCELLED`, `COMPLETED`)
- `page` (optional, default `1`): Page number
- `limit` (optional, default `10`): Items per page

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Trips retrieved successfully",
  "meta": {
    "totalItems": 4,
    "itemCount": 2,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  },
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440031",
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "startDate": "2026-04-10",
      "endDate": "2026-04-20",
      "minQuota": 5,
      "maxQuota": 25,
      "status": "ACTIVE",
      "pricings": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440041",
          "nationalityScope": "DOMESTIC",
          "basePrice": 32000000.00,
          "sellingPrice": 28000000.00
        },
        {
          "id": "550e8400-e29b-41d4-a716-446655440042",
          "nationalityScope": "INTERNATIONAL",
          "basePrice": 38000000.00,
          "sellingPrice": 34000000.00
        }
      ],
      "createdAt": "2026-09-04T10:00:00.000Z",
      "updatedAt": "2026-09-04T10:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440032",
      "variantId": "550e8400-e29b-41d4-a716-446655440020",
      "startDate": "2026-04-24",
      "endDate": "2026-05-04",
      "minQuota": 5,
      "maxQuota": 25,
      "status": "ACTIVE",
      "pricings": [
        {
          "id": "550e8400-e29b-41d4-a716-446655440043",
          "nationalityScope": "ALL",
          "basePrice": 32000000.00,
          "sellingPrice": 28000000.00
        }
      ],
      "createdAt": "2026-09-04T10:30:00.000Z",
      "updatedAt": "2026-09-04T10:30:00.000Z"
    }
  ]
}
```

---

### 3.3 Create Trip Window (`POST /api/v1/variants/:variantId/trips`)
```typescript
export class CreateTripDto {
  @IsDateString()
  startDate: string; // "YYYY-MM-DD", e.g. "2026-04-10"

  @IsDateString()
  endDate: string;   // "YYYY-MM-DD", e.g. "2026-04-20"

  @IsInt()
  @Min(1)
  minQuota: number = 1;

  @IsInt()
  @Min(1)
  maxQuota: number; // Total capacity, e.g. 25
}
```

---

### 3.4 List Trip Pricings (`GET /api/v1/trips/:tripId/pricings`)
Returns all pricing tiers for a specific trip departure.

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Trip pricings retrieved successfully",
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440041",
      "tripId": "550e8400-e29b-41d4-a716-446655440031",
      "nationalityScope": "DOMESTIC",
      "basePrice": 32000000.00,
      "sellingPrice": 28000000.00,
      "createdAt": "2026-09-04T10:00:00.000Z",
      "updatedAt": "2026-09-04T10:00:00.000Z"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440042",
      "tripId": "550e8400-e29b-41d4-a716-446655440031",
      "nationalityScope": "INTERNATIONAL",
      "basePrice": 38000000.00,
      "sellingPrice": 34000000.00,
      "createdAt": "2026-09-04T10:00:00.000Z",
      "updatedAt": "2026-09-04T10:00:00.000Z"
    }
  ]
}
```

---

### 3.5 Upsert Trip Pricing (`PUT /api/v1/trips/:tripId/pricings`)
```typescript
export class UpsertTripPricingDto {
  @IsString()
  @IsIn(['ALL', 'DOMESTIC', 'INTERNATIONAL'])
  nationalityScope: 'ALL' | 'DOMESTIC' | 'INTERNATIONAL' = 'ALL';

  @IsNumber()
  @Min(0)
  basePrice: number; // e.g. 32000000.00

  @IsNumber()
  @Min(0)
  sellingPrice: number; // e.g. 28000000.00
}
```

