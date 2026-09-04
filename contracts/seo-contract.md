# SEO & Metadata API Contracts

> **Overview**
> API contract specifications for polymorphic Search Engine Optimization (SEO), Open Graph metadata, and Schema.org structured data management across Products, Variants, and Destination Areas.
>
> **Related Design Document:** [SEO Technical Design](../technical/seo-technical-design.md)
> **Backend Guide:** [SEO Backend Guide](../backend/seo-backend-guide.md)
> **Frontend Guide:** [SEO Frontend Guide](../frontend/seo-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Polymorphic SEO** | `GET` | `/api/v1/seo/:targetType/:targetId` | Retrieve resolved SEO metadata (returns custom override if present, or dynamic formulaic fallback) |
| | `PUT` | `/api/v1/seo/:targetType/:targetId` | Create or update custom SEO metadata override for target entity |
| | `DELETE` | `/api/v1/seo/:targetType/:targetId` | Delete custom override, reverting target entity to dynamic formulaic fallback |
| **Sitemap Data** | `GET` | `/api/v1/seo/sitemap` | Paginated feed of active indexable URLs, last modified timestamps, and change frequencies |

---

## 1. Retrieve Resolved SEO Metadata (`GET /api/v1/seo/:targetType/:targetId`)

Returns the resolved SEO metadata for a specific entity. If no custom record exists in `seo_metadata`, the backend dynamically generates fallback values according to the formula matrix.

### Path Parameters
- `targetType`: Entity type — `PRODUCT | VARIANT | AREA`
- `targetId`: UUID of the entity

### Success Response (200 OK — Custom Override Present)

```json
{
  "statusCode": 200,
  "message": "SEO metadata retrieved successfully",
  "data": {
    "targetType": "VARIANT",
    "targetId": "550e8400-e29b-41d4-a716-446655440020",
    "isCustom": true,
    "metaTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
    "metaDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
    "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe/spring-2026",
    "ogTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
    "ogDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
    "ogImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-spring-hero.jpg",
    "noIndex": false,
    "noFollow": false,
    "structuredData": null,
    "createdAt": "2026-09-04T10:00:00.000Z",
    "updatedAt": "2026-09-04T10:00:00.000Z"
  }
}
```

### Success Response (200 OK — Dynamically Generated Fallback)

```json
{
  "statusCode": 200,
  "message": "Dynamic fallback SEO metadata resolved",
  "data": {
    "targetType": "VARIANT",
    "targetId": "550e8400-e29b-41d4-a716-446655440020",
    "isCustom": false,
    "metaTitle": "Grand West Europe (GWE) Spring 2026 (11D/10N) | Hobiholidays",
    "metaDescription": "Ikuti tour Grand West Europe (GWE) Spring 2026 mengunjungi Paris, Zurich, Amsterdam. Keberangkatan Apr 2026. Mulai IDR 34.500.000.",
    "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe/spring-2026",
    "ogTitle": "Grand West Europe (GWE) Spring 2026 (11D/10N) | Hobiholidays",
    "ogDescription": "Ikuti tour Grand West Europe (GWE) Spring 2026 mengunjungi Paris, Zurich, Amsterdam. Keberangkatan Apr 2026. Mulai IDR 34.500.000.",
    "ogImageUrl": "https://cdn.hobiholidays.com/defaults/og-tour-default.jpg",
    "noIndex": false,
    "noFollow": false,
    "structuredData": null
  }
}
```

---

## 2. Upsert Custom SEO Override (`PUT /api/v1/seo/:targetType/:targetId`)

Allows content editors or administrators to persist custom SEO meta tags, social sharing cards, and robot crawler directives.

### Request DTO (`UpsertSeoMetadataDto`)

```typescript
import {
  IsOptional,
  IsString,
  MaxLength,
  IsUrl,
  IsBoolean,
  IsObject,
} from 'class-validator';

export class UpsertSeoMetadataDto {
  @IsOptional()
  @IsString()
  @MaxLength(70, { message: 'metaTitle should not exceed 70 characters for optimal Google SERP display' })
  metaTitle?: string;

  @IsOptional()
  @IsString()
  @MaxLength(160, { message: 'metaDescription should not exceed 160 characters for optimal snippet display' })
  metaDescription?: string;

  @IsOptional()
  @IsUrl({}, { message: 'canonicalUrl must be a valid fully-qualified URL' })
  canonicalUrl?: string;

  @IsOptional()
  @IsString()
  @MaxLength(100)
  ogTitle?: string;

  @IsOptional()
  @IsString()
  @MaxLength(200)
  ogDescription?: string;

  @IsOptional()
  @IsUrl({}, { message: 'ogImageUrl must be a valid image URL' })
  ogImageUrl?: string;

  @IsOptional()
  @IsBoolean()
  noIndex?: boolean;

  @IsOptional()
  @IsBoolean()
  noFollow?: boolean;

  @IsOptional()
  @IsObject()
  structuredData?: Record<string, any>;
}
```

### Request Payload Example

```json
{
  "metaTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
  "metaDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
  "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe/spring-2026",
  "ogTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
  "ogDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
  "ogImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-spring-hero.jpg",
  "noIndex": false,
  "noFollow": false
}
```

### Success Response (200 OK)

```json
{
  "statusCode": 200,
  "message": "SEO metadata saved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440080",
    "targetType": "VARIANT",
    "targetId": "550e8400-e29b-41d4-a716-446655440020",
    "metaTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
    "metaDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
    "canonicalUrl": "https://www.hobiholidays.com/tours/grand-west-europe/spring-2026",
    "ogTitle": "Paket Tour Eropa Barat Musim Semi 2026 Murah | Hobiholidays",
    "ogDescription": "Liburan musim semi terbaik ke Eropa Barat bersama Hobiholidays. Kunjungi Paris, Swiss, dan Amsterdam selama 11 hari dengan penerbangan bintang 5.",
    "ogImageUrl": "https://cdn.hobiholidays.com/products/gwe/gwe-spring-hero.jpg",
    "noIndex": false,
    "noFollow": false,
    "structuredData": null,
    "createdAt": "2026-09-04T10:00:00.000Z",
    "updatedAt": "2026-09-04T10:00:00.000Z"
  }
}
```

---

## 3. Delete Custom SEO Override (`DELETE /api/v1/seo/:targetType/:targetId`)

Removes the custom `seo_metadata` row. Subsequent requests will return formulaic fallback values.

### Success Response (200 OK)

```json
{
  "statusCode": 200,
  "message": "Custom SEO metadata removed. Reverted to dynamic fallback generation.",
  "data": {
    "targetType": "VARIANT",
    "targetId": "550e8400-e29b-41d4-a716-446655440020",
    "deleted": true
  }
}
```

---

## 4. Error Responses (RFC 7807)

### 4.1 Validation Failure (400 Bad Request)

Emitted when title or description exceeds character boundaries, or URLs fail validation:

```json
{
  "statusCode": 400,
  "message": [
    "metaTitle should not exceed 70 characters for optimal Google SERP display",
    "ogImageUrl must be a valid image URL"
  ],
  "error": "Bad Request",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/seo/VARIANT/550e8400-e29b-41d4-a716-446655440020"
}
```

### 4.2 Target Entity Not Found (404 Not Found)

Emitted when attempting to attach SEO metadata to a nonexistent product, variant, or area:

```json
{
  "statusCode": 404,
  "message": "Target entity with type 'VARIANT' and id '550e8400-e29b-41d4-a716-446655440099' not found",
  "error": "Not Found",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/seo/VARIANT/550e8400-e29b-41d4-a716-446655440099"
}
```
