# API Contracts & Interface Specifications

> **Overview**
> Master directory and standards guide for all Hobiholidays REST API contracts. This specification defines endpoint routes, NestJS DTOs, request validation rules, and standard HTTP response envelopes across all domains.
>
> _Standardized on RESTful conventions, NestJS `class-validator`, and RFC 7807 problem details._

---

## 📚 Contracts Directory Map

| Contract Document | Domain Coverage | Target Technical Design | Backend Guide | Frontend Guide |
| :--- | :--- | :--- | :--- | :--- |
| **[Product Contracts](./product-contract.md)** | Product L1, Categories, Journeys, Split sub-resources (`/media`, `/locations`, `/variants`, `/supplementaries`, `/seo`), Variant Itineraries & Add-ons, Trips L3, Age-Band Pricings & Breakdown Components | [product-technical-design.md](../technical/product-technical-design.md) | [product-backend-guide.md](../backend/product-backend-guide.md) | [product-frontend-guide.md](../frontend/product-frontend-guide.md) |
| **[Product Hierarchy Contracts](./product-hierarchy-contract.md)** | All Tours listing feed, Variant card aggregation, Variant Detail page contract with embedded SEO | [product-hierarchy-technical-design.md](../technical/product-hierarchy-technical-design.md) | [product-hierarchy-backend-guide.md](../backend/product-hierarchy-backend-guide.md) | [product-hierarchy-frontend-guide.md](../frontend/product-hierarchy-frontend-guide.md) |
| **[Search & Filter Contracts](./product-search-filter-contract.md)** | Multi-criteria search query (`SearchTripDto`), Continent, Sub Continent, Country, POI, Category, Budget, Pack, Name, paginated response | [product-search-filter-technical-design.md](../technical/product-search-filter-technical-design.md) | [product-search-filter-backend-guide.md](../backend/product-search-filter-backend-guide.md) | [product-search-filter-frontend-guide.md](../frontend/product-search-filter-frontend-guide.md) |
| **[Product Media Contracts](./product-media-contract.md)** | Phase 1 multipart upload/streaming & Phase 2 presigned cloud uploads, polymorphic usages, brochure | [product-media-technical-design.md](../technical/product-media-technical-design.md) | [product-media-backend-guide.md](../backend/product-media-backend-guide.md) | [product-media-frontend-guide.md](../frontend/product-media-frontend-guide.md) |
| **[Area Domain Contracts](./area-contract.md)** | Area types, Continent/Sub Continent/Country/POI 4-tier hierarchy, autocomplete, search, administrative CRUD | [area-technical-design.md](../technical/area-technical-design.md) | [area-backend-guide.md](../backend/area-backend-guide.md) | [area-frontend-guide.md](../frontend/area-frontend-guide.md) |
| **[SEO Contracts](./seo-contract.md)** | Polymorphic SEO overrides (`/api/v1/seo/:targetType/:targetId`), dynamic fallbacks, social metadata | [seo-technical-design.md](../technical/seo-technical-design.md) | [seo-backend-guide.md](../backend/seo-backend-guide.md) | [seo-frontend-guide.md](../frontend/seo-frontend-guide.md) |

---

## 🌐 Global API Standards

### 1. Base URL & Versioning
All API endpoints follow semantic URL versioning:
- **Base URL:** `https://api.hobiholidays.com/api/v1`
- **Content-Type:** `application/json` (except multipart upload endpoints which use `multipart/form-data`)

### 2. Standard Success Response Envelope
All single-item endpoints return a standard data envelope:

```typescript
export interface ApiResponse<T> {
  statusCode: number; // e.g. 200, 201
  message: string;    // Human-readable status (e.g. "Product retrieved successfully")
  data: T;            // Response payload
}
```

### 3. Standard Paginated Response Envelope
List and search endpoints return a metadata pagination block:

```typescript
export interface PaginatedApiResponse<T> {
  statusCode: number;
  message: string;
  meta: {
    totalItems: number;   // Total records matching the query
    itemCount: number;    // Number of records on current page
    itemsPerPage: number; // Limit per page
    totalPages: number;   // Total pages available
    currentPage: number;  // Active page (1-indexed)
  };
  data: T[];
}
```

### 4. Standard Error Envelope (RFC 7807)
Failed requests emit a consistent error shape:

```typescript
export interface ApiErrorResponse {
  statusCode: number;    // e.g. 400, 404, 409, 422, 500
  message: string | string[]; // Error description or array of validation errors
  error: string;         // HTTP status text (e.g. "Bad Request")
  timestamp: string;     // ISO 8601 string
  path: string;          // Request path
}
```

- **Example 400 Bad Request Response:**
```json
{
  "statusCode": 400,
  "message": [
    "totalPack must not be less than 1",
    "departureMonth must match format YYYY-MM"
  ],
  "error": "Bad Request",
  "timestamp": "2026-09-04T10:30:00.000Z",
  "path": "/api/v1/variants/search"
}
```
