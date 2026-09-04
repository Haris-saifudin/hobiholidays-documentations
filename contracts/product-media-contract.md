# Product Media API Contracts

> **Overview**
> API contract specifications for the Product Media subsystem, supporting the **2-Phase Progressive Storage Strategy**:
> - **Phase 1 (Database-First):** Multipart uploads to backend and direct streaming from PostgreSQL blobs.
> - **Phase 2 (Cloud Storage & CDN):** Presigned S3/R2 upload URLs and edge CDN delivery.
> - **Usage Management:** Polymorphic attachment to Products, Variants, and Itinerary items, ordered gallery reordering, and atomic 1:1 Itinerary PDF brochure assignment.
>
> **Related Design Document:** [Product Media Technical Design](../technical/product-media-technical-design.md)
> **Backend Guide:** [Product Media Backend Guide](../backend/product-media-backend-guide.md)
> **Frontend Guide:** [Product Media Frontend Guide](../frontend/product-media-frontend-guide.md)

---

## 📑 Endpoints Summary Table

| Phase / Category | Method | Endpoint | Description |
| :--- | :--- | :--- | :--- |
| **Phase 1 (Database)** | `POST` | `/api/v1/products/:productId/media/upload` | Multipart file upload stored in PostgreSQL |
| | `GET` | `/api/v1/media/:id/stream` | Stream binary image/video with HTTP caching |
| | `GET` | `/api/v1/media/:id/download` | Download PDF brochure with attachment header |
| **Phase 2 (Cloud)** | `POST` | `/api/v1/products/:productId/media/presigned-url` | Generate S3/R2 direct presigned upload URL |
| | `POST` | `/api/v1/products/:productId/media` | Register uploaded cloud asset in database |
| **Usages (Both Phases)**| `POST` | `/api/v1/products/:productId/media/usages` | Attach media to Product, Variant, or Itinerary Item |
| | `PUT` | `/api/v1/products/:productId/media/usages/reorder`| Reorder gallery photos |
| | `DELETE`| `/api/v1/products/:productId/media/usages/:usageId`| Detach media from usage slot |
| **Itinerary Brochure** | `PUT` | `/api/v1/products/:productId/itinerary-pdf` | Atomically assign single official itinerary PDF |

---

## 1. Phase 1: Database-First Endpoints

### 1.1 Multipart Upload (`POST /api/v1/products/:productId/media/upload`)
Receives binary file upload via standard `multipart/form-data`. Saves metadata to `product_media` and raw bytes to `product_media_blobs`.

#### Request
- **Headers:** `Content-Type: multipart/form-data`
- **Form Fields:**
  - `file`: Binary file buffer (max 25MB)
  - `mediaType`: `'IMAGE' | 'VIDEO' | 'PDF'`

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Media uploaded successfully to database",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440050",
    "productId": "550e8400-e29b-41d4-a716-446655440010",
    "storageProvider": "DATABASE",
    "mediaType": "IMAGE",
    "fileName": "gwe-hero-paris.jpg",
    "fileSizeBytes": 2410500,
    "mimeType": "image/jpeg",
    "url": "/api/v1/media/550e8400-e29b-41d4-a716-446655440050/stream",
    "createdAt": "2026-09-04T10:00:00.000Z"
  }
}
```

---

### 1.2 Binary Streaming (`GET /api/v1/media/:id/stream`)
Streams binary image or video with aggressive HTTP caching headers.

#### Response Headers
```http
HTTP/1.1 200 OK
Content-Type: image/jpeg
Content-Length: 2410500
Cache-Control: public, max-age=31536000, immutable
ETag: "550e8400-e29b-41d4-a716-446655440050-1785313920"
```

---

### 1.3 PDF Brochure Download (`GET /api/v1/media/:id/download`)
Streams PDF document with forced browser download prompt.

#### Response Headers
```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Length: 4613734
Content-Disposition: attachment; filename="Grand-West-Europe-Brochure-2026.pdf"
Cache-Control: public, max-age=86400
```

---

## 2. Phase 2: Cloud Object Storage Endpoints

### 2.1 Request Presigned Upload URL (`POST /api/v1/products/:productId/media/presigned-url`)

#### Request DTO
```typescript
import { IsString, IsIn, IsInt, Min, Max } from 'class-validator';

export class GeneratePresignedUrlDto {
  @IsString()
  fileName: string; // e.g. "eiffel-tower.jpg"

  @IsString()
  @IsIn([
    'image/jpeg',
    'image/png',
    'image/webp',
    'video/mp4',
    'application/pdf',
  ])
  mimeType: string;

  @IsInt()
  @Min(1)
  @Max(52428800) // 50MB
  fileSizeBytes: number;

  @IsString()
  @IsIn(['IMAGE', 'VIDEO', 'PDF'])
  mediaType: 'IMAGE' | 'VIDEO' | 'PDF';
}
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Presigned upload URL generated",
  "data": {
    "uploadUrl": "https://hobiholidays-bucket.s3.ap-southeast-1.amazonaws.com/products/prod_gwe_01/media/eiffel-tower.jpg?X-Amz-Signature=...",
    "objectKey": "products/prod_gwe_01/media/eiffel-tower.jpg",
    "cdnUrl": "https://cdn.hobiholidays.com/products/prod_gwe_01/media/eiffel-tower.jpg",
    "expiresInSeconds": 900
  }
}
```

---

### 2.2 Register Cloud Asset (`POST /api/v1/products/:productId/media`)
Registers asset after browser finishes direct PUT upload to S3/R2.

#### Request DTO
```typescript
import { IsString, IsIn, IsInt, IsUrl, IsOptional } from 'class-validator';

export class CreateCloudMediaDto {
  @IsString()
  @IsIn(['IMAGE', 'VIDEO', 'PDF'])
  mediaType: 'IMAGE' | 'VIDEO' | 'PDF';

  @IsString()
  fileName: string;

  @IsInt()
  fileSizeBytes: number;

  @IsString()
  mimeType: string;

  @IsString()
  objectKey: string;

  @IsUrl()
  url: string; // Full CDN URL

  @IsOptional()
  @IsString()
  sourceUploadId?: string;
}
```

---

## 3. Polymorphic Usage Endpoints (Both Phases)

### 3.1 Attach Media Usage (`POST /api/v1/products/:productId/media/usages`)

#### Request DTO
```typescript
import { IsUUID, IsString, IsIn, IsInt, Min } from 'class-validator';

export class AttachMediaUsageDto {
  @IsUUID('4')
  mediaId: string;

  @IsString()
  @IsIn(['PRODUCT', 'VARIANT', 'ITINERARY_ITEM'])
  targetType: 'PRODUCT' | 'VARIANT' | 'ITINERARY_ITEM';

  @IsUUID('4')
  targetId: string;

  @IsString()
  @IsIn(['COVER', 'GALLERY', 'THUMBNAIL', 'ITINERARY_PDF', 'ATTACHMENT'])
  usageContext: 'COVER' | 'GALLERY' | 'THUMBNAIL' | 'ITINERARY_PDF' | 'ATTACHMENT';

  @IsInt()
  @Min(0)
  sortOrder: number = 0;
}
```

#### Success Response (201 Created)
```json
{
  "statusCode": 201,
  "message": "Media usage attached successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440060",
    "mediaId": "550e8400-e29b-41d4-a716-446655440050",
    "targetType": "PRODUCT",
    "targetId": "550e8400-e29b-41d4-a716-446655440010",
    "usageContext": "COVER",
    "sortOrder": 0
  }
}
```

---

### 3.2 Reorder Gallery Usages (`PUT /api/v1/products/:productId/media/usages/reorder`)

#### Request DTO
```typescript
import { IsArray, ValidateNested, IsUUID, IsInt, Min } from 'class-validator';
import { Type } from 'class-transformer';

class UsageOrderItem {
  @IsUUID('4')
  usageId: string;

  @IsInt()
  @Min(0)
  sortOrder: number;
}

export class ReorderMediaUsagesDto {
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => UsageOrderItem)
  orders: UsageOrderItem[];
}
```

---

## 4. Official Itinerary PDF Brochure (`PUT /api/v1/products/:productId/itinerary-pdf`)

Guarantees atomic attachment of a single official downloadable brochure:
1. Validates `media_type = 'PDF'`.
2. Upserts `product_media_usages` with `usage_context = 'ITINERARY_PDF'` (guaranteed unique by `uq_media_usages_product_itinerary_pdf`).
3. Updates `products.itinerary_pdf_url = media.url`.

#### Request DTO
```typescript
import { IsUUID } from 'class-validator';

export class SetItineraryPdfDto {
  @IsUUID('4')
  mediaId: string;
}
```

#### Success Response (200 OK)
```json
{
  "statusCode": 200,
  "message": "Official itinerary PDF brochure updated successfully",
  "data": {
    "productId": "550e8400-e29b-41d4-a716-446655440010",
    "itineraryPdfUrl": "https://cdn.hobiholidays.com/docs/itineraries/gwe-brochure.pdf",
    "fileName": "GWE-Brochure-2026.pdf",
    "fileSizeBytes": 4613734
  }
}
```
