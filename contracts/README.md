# API Contracts & Interface Specifications

> **Overview**
> Master directory and standards guide for all Hobiholidays REST API contracts. This specification defines endpoint routes, NestJS DTOs, request validation rules, and standard HTTP response envelopes across all domains.
>
> _Standardized on RESTful conventions, NestJS `class-validator`, and RFC 7807 problem details._

---

## 📚 Contracts Directory Map

| Contract Document | Domain Coverage | Target Technical Design |
| :--- | :--- | :--- |
| **[Product Contracts](./product-contract.md)** | Product L1, Journeys, Split sub-resources (`/media`, `/itineraries`, `/locations`, `/variants`, `/supplementaries`, `/seo`), Trips L3, Pricings | [product-technical-design.md](../product-technical-design.md) |
| **[Product Hierarchy Contracts](./product-hierarchy-contract.md)** | All Tours listing feed, Variant card aggregation, Variant Detail page contract with embedded SEO | [product-hierarchy-technical-design.md](../product-hierarchy-technical-design.md) |
| **[Search & Filter Contracts](./product-search-filter-contract.md)** | Multi-criteria search query (`SearchTripDto`), Continent, Country, Budget, Pack, Name, paginated response | [product-search-filter-technical-design.md](../product-search-filter-technical-design.md) |
| **[Product Media Contracts](./product-media-contract.md)** | Phase 1 multipart upload/streaming & Phase 2 presigned cloud uploads, polymorphic usages, brochure | [product-media-technical-design.md](../product-media-technical-design.md) |
| **[Area Domain Contracts](./area-contract.md)** | Area types, Continent/Country/City hierarchy, autocomplete, search, administrative CRUD | [area-technical-design.md](../area-technical-design.md) |

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
