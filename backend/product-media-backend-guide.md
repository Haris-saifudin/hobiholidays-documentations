# Product Media Subsystem — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend engineering guide for the Product Media and Binary Asset subsystem, implementing the **2-Phase Progressive Storage Strategy**:
> - **Phase 1 (Database-First):** Multipart uploads to backend, raw `BYTEA` storage in `product_media_blobs`, and high-performance streaming (`/api/v1/media/:id/stream`) with immutable caching headers.
> - **Phase 2 (Cloud Scale):** AWS S3 / Cloudflare R2 presigned upload generation and CDN delivery.
> - **Zero-Downtime Migration:** Batch script to transfer Phase 1 database blobs to cloud storage with zero frontend breaking changes.
>
> **Related Design Document:** [Product Media Technical Design](../technical/product-media-technical-design.md)
> **API Contract:** [Product Media Contracts](../contracts/product-media-contract.md)
> **Frontend Guide:** [Product Media Frontend Guide](../frontend/product-media-frontend-guide.md)

---

## 🏗️ Module Architecture

```
src/modules/media/
├── media.module.ts
├── controllers/
│   ├── product-media.controller.ts       # Uploads, presigned URLs, and usages
│   └── media-stream.controller.ts        # Phase 1 binary streaming & PDF download
├── services/
│   ├── product-media.service.ts          # Metadata persistence & usage bindings
│   ├── database-blob.service.ts          # Phase 1 BYTEA read/write
│   └── s3-storage.service.ts             # Phase 2 AWS SDK presigned URLs
└── scripts/
    └── migrate-media-to-cloud.ts         # Phase 1 -> Phase 2 cloud migration runner
```

---

## 💾 Phase 1: Database-First Streaming & Binary Storage

### 1. Multipart Upload Controller (`ProductMediaController`)

```typescript
// controllers/product-media.controller.ts
import {
  Controller,
  Post,
  Param,
  UseInterceptors,
  UploadedFile,
  Body,
  ParseUUIDPipe,
  HttpStatus,
  BadRequestException,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { ProductMediaService } from '../services/product-media.service';
import { UploadMediaDto } from '../dto/upload-media.dto';

@Controller('products/:productId/media')
export class ProductMediaController {
  constructor(private readonly mediaService: ProductMediaService) {}

  @Post('upload')
  @UseInterceptors(
    FileInterceptor('file', {
      limits: { fileSize: 25 * 1024 * 1024 }, // 25 MB max limit
      fileFilter: (req, file, cb) => {
        const allowedMimes = [
          'image/jpeg', 'image/png', 'image/webp',
          'application/pdf', 'video/mp4',
        ];
        if (allowedMimes.includes(file.mimetype)) {
          cb(null, true);
        } else {
          cb(new BadRequestException(`Unsupported file type: ${file.mimetype}`), false);
        }
      },
    }),
  )
  async uploadMedia(
    @Param('productId', ParseUUIDPipe) productId: string,
    @UploadedFile() file: Express.Multer.File,
    @Body() dto: UploadMediaDto,
  ) {
    if (!file) {
      throw new BadRequestException('File is required');
    }

    const media = await this.mediaService.saveBinaryFile(productId, file, dto.mediaType);

    return {
      statusCode: HttpStatus.CREATED,
      message: 'Media uploaded successfully to database',
      data: media,
    };
  }
}
```

### 2. High-Performance Streaming Controller (`MediaStreamController`)

Streams raw bytes directly from PostgreSQL `product_media_blobs` with aggressive browser caching headers:

```typescript
// controllers/media-stream.controller.ts
import { Controller, Get, Param, Res, ParseUUIDPipe, NotFoundException } from '@nestjs/common';
import { Response } from 'express';
import { ProductMediaService } from '../services/product-media.service';

@Controller('media')
export class MediaStreamController {
  constructor(private readonly mediaService: ProductMediaService) {}

  /**
   * Streams image or video with immutable 1-year browser cache
   */
  @Get(':id/stream')
  async streamMedia(
    @Param('id', ParseUUIDPipe) id: string,
    @Res() res: Response,
  ) {
    const { metadata, blob } = await this.mediaService.getMediaWithBlob(id);

    if (!blob || !blob.file_data) {
      throw new NotFoundException('Binary data not found for this media item');
    }

    res.set({
      'Content-Type': metadata.mime_type,
      'Content-Length': metadata.file_size_bytes,
      'Cache-Control': 'public, max-age=31536000, immutable',
      'ETag': `"${metadata.id}-${new Date(metadata.updated_at).getTime()}"`,
    });

    return res.send(blob.file_data);
  }

  /**
   * Serves PDF brochure download with Content-Disposition attachment header
   */
  @Get(':id/download')
  async downloadBrochure(
    @Param('id', ParseUUIDPipe) id: string,
    @Res() res: Response,
  ) {
    const { metadata, blob } = await this.mediaService.getMediaWithBlob(id);

    if (!blob || !blob.file_data) {
      throw new NotFoundException('PDF binary data not found');
    }

    res.set({
      'Content-Type': 'application/pdf',
      'Content-Length': metadata.file_size_bytes,
      'Content-Disposition': `attachment; filename="${encodeURIComponent(metadata.file_name)}"`,
      'Cache-Control': 'public, max-age=86400',
    });

    return res.send(blob.file_data);
  }
}
```

### 3. Media Subsystem Service (`ProductMediaService`)

Handles atomic database transactions for saving binary buffers, usage bindings, and the strict 1:1 itinerary PDF brochure guarantee:

```typescript
// services/product-media.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class ProductMediaService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Phase 1: Saves metadata to product_media and raw bytes to product_media_blobs.
   */
  async saveBinaryFile(productId: string, file: Express.Multer.File, mediaType: string) {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      const mediaResult = await queryRunner.query(`
        INSERT INTO product_media (
          product_id, storage_provider, media_type, file_name,
          file_size_bytes, mime_type, url
        ) VALUES ($1, 'DATABASE', $2, $3, $4, $5, '')
        RETURNING id;
      `, [productId, mediaType, file.originalname, file.size, file.mimetype]);

      const mediaId = mediaResult[0].id;
      const streamUrl = `/api/v1/media/${mediaId}/stream`;

      // Update URL with generated stream endpoint
      await queryRunner.query(
        `UPDATE product_media SET url = $1 WHERE id = $2`,
        [streamUrl, mediaId],
      );

      // Insert binary bytes into blobs isolation table
      await queryRunner.query(`
        INSERT INTO product_media_blobs (media_id, file_data)
        VALUES ($1, $2);
      `, [mediaId, file.buffer]);

      await queryRunner.commitTransaction();

      return {
        id: mediaId,
        productId,
        storageProvider: 'DATABASE',
        mediaType,
        fileName: file.originalname,
        fileSizeBytes: file.size,
        mimeType: file.mimetype,
        url: streamUrl,
      };
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw new BadRequestException(`Failed to save media: ${err.message}`);
    } finally {
      await queryRunner.release();
    }
  }

  /**
   * Fetches metadata and binary bytes for streaming or download.
   */
  async getMediaWithBlob(id: string) {
    const metaRows = await this.dataSource.query(
      `SELECT * FROM product_media WHERE id = $1`,
      [id],
    );
    if (!metaRows.length) {
      throw new NotFoundException(`Media '${id}' not found`);
    }

    const blobRows = await this.dataSource.query(
      `SELECT file_data FROM product_media_blobs WHERE media_id = $1`,
      [id],
    );

    return {
      metadata: metaRows[0],
      blob: blobRows[0] || null,
    };
  }

  /**
   * Attaches media to a target entity (Product, Variant, Itinerary Item)
   */
  async attachUsage(dto: {
    mediaId: string;
    targetType: 'PRODUCT' | 'VARIANT' | 'ITINERARY_ITEM';
    targetId: string;
    usageContext: 'COVER' | 'GALLERY' | 'THUMBNAIL' | 'ITINERARY_PDF' | 'ATTACHMENT';
    sortOrder?: number;
  }) {
    const result = await this.dataSource.query(`
      INSERT INTO product_media_usages (
        media_id, target_type, target_id, usage_context, sort_order
      ) VALUES ($1, $2, $3, $4, $5)
      RETURNING id, media_id, target_type, target_id, usage_context, sort_order;
    `, [dto.mediaId, dto.targetType, dto.targetId, dto.usageContext, dto.sortOrder || 0]);

    return result[0];
  }

  /**
   * Sets the single official itinerary PDF brochure and synchronizes products.itinerary_pdf_url.
   */
  async setItineraryPdf(productId: string, mediaId: string) {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Verify media exists and is PDF
      const mediaRows = await queryRunner.query(
        `SELECT id, url, media_type FROM product_media WHERE id = $1 AND product_id = $2`,
        [mediaId, productId],
      );
      if (!mediaRows.length || mediaRows[0].media_type !== 'PDF') {
        throw new BadRequestException('Selected media must be a valid PDF document belonging to this product');
      }

      // 2. Remove existing ITINERARY_PDF usage for this product
      await queryRunner.query(
        `DELETE FROM product_media_usages WHERE target_type = 'PRODUCT' AND target_id = $1 AND usage_context = 'ITINERARY_PDF'`,
        [productId],
      );

      // 3. Insert new 1:1 ITINERARY_PDF usage
      await queryRunner.query(
        `INSERT INTO product_media_usages (media_id, target_type, target_id, usage_context, sort_order)
         VALUES ($1, 'PRODUCT', $2, 'ITINERARY_PDF', 0)`,
        [mediaId, productId],
      );

      // 4. Update denormalized fast-read column on products
      const pdfUrl = mediaRows[0].url;
      await queryRunner.query(
        `UPDATE products SET itinerary_pdf_url = $1, updated_at = NOW() WHERE id = $2`,
        [pdfUrl, productId],
      );

      await queryRunner.commitTransaction();

      return {
        productId,
        mediaId,
        itineraryPdfUrl: pdfUrl,
      };
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw err;
    } finally {
      await queryRunner.release();
    }
  }
}
```

---

## ☁️ Phase 2: AWS S3 / Cloudflare R2 Presigned Uploads

When cloud storage is active, clients upload directly to S3 via presigned PUT URLs, bypassing the backend:

```typescript
// services/s3-storage.service.ts
import { Injectable } from '@nestjs/common';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

@Injectable()
export class S3StorageService {
  private readonly s3Client: S3Client;
  private readonly bucketName: string;
  private readonly cdnBaseUrl: string;

  constructor() {
    this.s3Client = new S3Client({
      region: process.env.AWS_REGION || 'ap-southeast-1',
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || '',
      },
    });
    this.bucketName = process.env.AWS_S3_BUCKET || 'hobiholidays-media';
    this.cdnBaseUrl = process.env.CDN_BASE_URL || 'https://cdn.hobiholidays.com';
  }

  async generatePresignedUploadUrl(
    productId: string,
    fileName: string,
    mimeType: string,
  ) {
    const sanitizedName = fileName.replace(/[^a-zA-Z0-9.-]/g, '_');
    const objectKey = `products/${productId}/${Date.now()}-${sanitizedName}`;

    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: objectKey,
      ContentType: mimeType,
    });

    const uploadUrl = await getSignedUrl(this.s3Client, command, { expiresIn: 900 }); // 15 mins

    return {
      uploadUrl,
      objectKey,
      cdnUrl: `${this.cdnBaseUrl}/${objectKey}`,
    };
  }
}
```

---

## 🚀 Zero-Downtime Phase 1 to Phase 2 Migration Script

When transitioning from database storage to cloud object storage:

```typescript
// scripts/migrate-media-to-cloud.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { Pool } from 'pg';

const s3 = new S3Client({ region: process.env.AWS_REGION || 'ap-southeast-1' });
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const CDN_BASE_URL = process.env.CDN_BASE_URL || 'https://cdn.hobiholidays.com';
const BUCKET_NAME = process.env.AWS_S3_BUCKET || 'hobiholidays-production-bucket';

export async function migrateMediaToCloud() {
  const client = await pool.connect();
  try {
    const { rows } = await client.query(`
      SELECT m.id, m.product_id, m.file_name, m.mime_type, b.file_data
      FROM product_media m
      INNER JOIN product_media_blobs b ON b.media_id = m.id
      WHERE m.storage_provider = 'DATABASE'
    `);

    console.log(`Starting migration of ${rows.length} assets to Cloud Storage...`);

    for (const media of rows) {
      const objectKey = `products/${media.product_id}/${media.id}/${media.file_name}`;
      const cdnUrl = `${CDN_BASE_URL}/${objectKey}`;

      // 1. Upload binary buffer to S3 / Cloudflare R2
      await s3.send(
        new PutObjectCommand({
          Bucket: BUCKET_NAME,
          Key: objectKey,
          Body: media.file_data,
          ContentType: media.mime_type,
        }),
      );

      // 2. Update metadata in database
      await client.query(`
        UPDATE product_media
        SET storage_provider = 'S3', object_key = $1, url = $2, updated_at = NOW()
        WHERE id = $3
      `, [objectKey, cdnUrl, media.id]);

      // 3. Update denormalized itinerary_pdf_url if applicable
      await client.query(`
        UPDATE products
        SET itinerary_pdf_url = $1
        WHERE id = $2 AND itinerary_pdf_url LIKE '%/api/v1/media/' || $3 || '/%'
      `, [cdnUrl, media.product_id, media.id]);

      // 4. Delete binary data from database to reclaim disk space
      await client.query(`DELETE FROM product_media_blobs WHERE media_id = $1`, [media.id]);

      console.log(`✓ Migrated: ${media.file_name} -> ${cdnUrl}`);
    }

    console.log('Media migration completed successfully.');
  } finally {
    client.release();
  }
}
```
