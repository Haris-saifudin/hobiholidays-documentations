# Product Domain — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Implementation guide for the Product Domain backend service. Covers `ProductModule`, split sub-resource controller architecture, transactional creation of Products and Journeys, status lifecycle state validation, and cascade soft-deletion.
>
> **Related Design Document:** [Product Technical Design](../technical/product-technical-design.md)
> **API Contract:** [Product Contracts](../contracts/product-contract.md)
> **Frontend Guide:** [Product Frontend Guide](../frontend/product-frontend-guide.md)

---

## 🏗️ Module Architecture

The Product domain is organized under `src/modules/product/`:

```
src/modules/product/
├── product.module.ts
├── controllers/
│   ├── product.controller.ts             # Base product CRUD & split sub-resources
│   └── product-subresource.controller.ts # Dedicated sub-resource mutations
├── services/
│   ├── product.service.ts                # Master product lifecycle & transactions
│   ├── product-itinerary.service.ts      # Itinerary & day-by-day item handling
│   ├── product-location.service.ts       # Destination marker attachment
│   └── product-supplementary.service.ts  # Inclusions, exclusions, notes
└── entities/                             # TypeORM / Kysely schemas
    ├── product.entity.ts
    ├── product-journey.entity.ts
    └── ...
```

### Module Definition

```typescript
// product.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ProductController } from './controllers/product.controller';
import { ProductService } from './services/product.service';
import { ProductItineraryService } from './services/product-itinerary.service';
import { ProductLocationService } from './services/product-location.service';
import { ProductSupplementaryService } from './services/product-supplementary.service';
import { Product } from './entities/product.entity';
import { ProductJourney } from './entities/product-journey.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([
      Product,
      ProductJourney,
    ]),
  ],
  controllers: [ProductController],
  providers: [
    ProductService,
    ProductItineraryService,
    ProductLocationService,
    ProductSupplementaryService,
  ],
  exports: [ProductService],
})
export class ProductModule {}
```

---

## 🎮 Controller Implementation

The controller provides base product CRUD operations and the granular split sub-resource endpoints designed to eliminate over-fetching in micro-frontends:

```typescript
// controllers/product.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Param,
  Body,
  Query,
  ParseUUIDPipe,
  HttpStatus,
  HttpCode,
} from '@nestjs/common';
import { ProductService } from '../services/product.service';
import { CreateProductDto } from '../dto/create-product.dto';
import { UpdateProductDto } from '../dto/update-product.dto';
import { ListProductsDto } from '../dto/list-products.dto';

@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  // =========================================================================
  // 1. BASE PRODUCT ENDPOINTS
  // =========================================================================

  @Get()
  async listProducts(@Query() query: ListProductsDto) {
    const { data, meta } = await this.productService.list(query);
    return {
      statusCode: HttpStatus.OK,
      message: 'Products retrieved successfully',
      meta,
      data,
    };
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async createProduct(@Body() dto: CreateProductDto) {
    const data = await this.productService.create(dto);
    return {
      statusCode: HttpStatus.CREATED,
      message: 'Product created successfully',
      data,
    };
  }

  @Get(':id')
  async getProduct(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.findById(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product retrieved successfully',
      data,
    };
  }

  @Put(':id')
  async updateProduct(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateProductDto,
  ) {
    const data = await this.productService.update(id, dto);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product updated successfully',
      data,
    };
  }

  @Delete(':id')
  async deleteProduct(@Param('id', ParseUUIDPipe) id: string) {
    await this.productService.softDelete(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product deleted successfully',
      data: { id, deleted: true },
    };
  }

  // =========================================================================
  // 2. SPLIT SUB-RESOURCE RETRIEVAL ENDPOINTS
  // =========================================================================

  @Get(':id/media')
  async getProductMedia(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getMedia(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product media retrieved successfully',
      data,
    };
  }

  @Get(':id/itineraries')
  async getProductItineraries(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getItineraries(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product itineraries retrieved successfully',
      data,
    };
  }

  @Get(':id/locations')
  async getProductLocations(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getLocations(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product locations retrieved successfully',
      data,
    };
  }

  @Get(':id/variants')
  async getProductVariants(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getVariants(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product variants retrieved successfully',
      data,
    };
  }

  @Get(':id/supplementaries')
  async getProductSupplementaries(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getSupplementaries(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product supplementaries retrieved successfully',
      data,
    };
  }

  @Get(':id/seo')
  async getProductSeo(@Param('id', ParseUUIDPipe) id: string) {
    const data = await this.productService.getSeo(id);
    return {
      statusCode: HttpStatus.OK,
      message: 'Product SEO retrieved successfully',
      data,
    };
  }
}
```

---

## ⚙️ Service Implementation & Atomic Transactions

Creating a Product requires atomically generating the default journey duration in `product_journeys`:

```typescript
// services/product.service.ts
import { Injectable, NotFoundException, BadRequestException } from '@nestjs/common';
import { DataSource, Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';
import { Product } from '../entities/product.entity';
import { ProductJourney } from '../entities/product-journey.entity';
import { CreateProductDto } from '../dto/create-product.dto';
import { ListProductsDto } from '../dto/list-products.dto';

@Injectable()
export class ProductService {
  constructor(
    @InjectRepository(Product)
    private readonly productRepo: Repository<Product>,
    private readonly dataSource: DataSource,
  ) {}

  /**
   * Atomic Product Creation:
   * Inserts the master product record and its baseline journey duration within a single transaction.
   */
  async create(dto: CreateProductDto) {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Insert master Product
      const product = queryRunner.manager.create(Product, {
        code: dto.code,
        name: dto.name,
        slug: dto.slug,
        productType: dto.productType,
        headline: dto.headline,
        description: dto.description,
        listingStatus: dto.listingStatus || 'DRAFT',
      });
      const savedProduct = await queryRunner.manager.save(product);

      // 2. Insert baseline ProductJourney duration
      const journey = queryRunner.manager.create(ProductJourney, {
        productId: savedProduct.id,
        durationDays: dto.durationDays,
        durationNights: dto.durationNights,
        nationalityScope: 'ALL',
      });
      await queryRunner.manager.save(journey);

      await queryRunner.commitTransaction();

      return {
        ...savedProduct,
        durationDays: dto.durationDays,
        durationNights: dto.durationNights,
      };
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw new BadRequestException(`Failed to create product: ${error.message}`);
    } finally {
      await queryRunner.release();
    }
  }

  /**
   * Paginated Product Listing with filtering
   */
  async list(dto: ListProductsDto) {
    const page = dto.page || 1;
    const limit = dto.limit || 10;
    const skip = (page - 1) * limit;

    const qb = this.productRepo.createQueryBuilder('p')
      .leftJoinAndSelect('p.journeys', 'j')
      .where('p.deletedAt IS NULL');

    if (dto.listingStatus) {
      qb.andWhere('p.listingStatus = :status', { status: dto.listingStatus });
    }
    if (dto.productType) {
      qb.andWhere('p.productType = :type', { type: dto.productType });
    }
    if (dto.search) {
      qb.andWhere('(p.name ILIKE :search OR p.code ILIKE :search)', {
        search: `%${dto.search}%`,
      });
    }

    const [items, totalItems] = await qb
      .skip(skip)
      .take(limit)
      .orderBy('p.createdAt', 'DESC')
      .getManyAndCount();

    const totalPages = Math.ceil(totalItems / limit);

    return {
      data: items,
      meta: {
        totalItems,
        itemCount: items.length,
        itemsPerPage: limit,
        totalPages,
        currentPage: page,
      },
    };
  }

  async findById(id: string) {
    const product = await this.productRepo.findOne({
      where: { id, deletedAt: null as any },
      relations: ['journeys'],
    });
    if (!product) {
      throw new NotFoundException(`Product with ID '${id}' not found`);
    }
    return product;
  }

  async softDelete(id: string) {
    const product = await this.findById(id);
    product.deletedAt = new Date();
    product.listingStatus = 'ARCHIVED';
    return this.productRepo.save(product);
  }

  // =========================================================================
  // Sub-resource dedicated queries
  // =========================================================================

  async getMedia(productId: string) {
    await this.findById(productId);
    const usages = await this.dataSource.query(`
      SELECT m.id AS media_id, m.media_type, m.file_name, m.file_size_bytes, m.mime_type, m.url,
             u.id AS usage_id, u.target_type, u.usage_context, u.sort_order
      FROM product_media m
      INNER JOIN product_media_usages u ON u.media_id = m.id
      WHERE m.product_id = $1
      ORDER BY u.sort_order ASC
    `, [productId]);

    const coverRow = usages.find((u: any) => u.usage_context === 'COVER');
    const pdfRow = usages.find((u: any) => u.usage_context === 'ITINERARY_PDF');
    const galleryRows = usages.filter((u: any) => u.usage_context === 'GALLERY');

    return {
      productId,
      cover: coverRow ? {
        mediaId: coverRow.media_id,
        url: coverRow.url,
        fileName: coverRow.file_name,
        fileSizeBytes: parseInt(coverRow.file_size_bytes, 10),
        mimeType: coverRow.mime_type,
      } : null,
      gallery: galleryRows.map((g: any) => ({
        usageId: g.usage_id,
        mediaId: g.media_id,
        url: g.url,
        fileName: g.file_name,
        sortOrder: g.sort_order,
      })),
      itineraryPdf: pdfRow ? {
        mediaId: pdfRow.media_id,
        url: pdfRow.url,
        fileName: pdfRow.file_name,
        fileSizeBytes: parseInt(pdfRow.file_size_bytes, 10),
        mimeType: pdfRow.mime_type,
      } : null,
    };
  }

  async getItineraries(productId: string) {
    const product = await this.findById(productId);
    const itineraryRows = await this.dataSource.query(`
      SELECT id, source_type, itinerary_type
      FROM product_itineraries
      WHERE product_id = $1
      LIMIT 1
    `, [productId]);

    if (!itineraryRows.length) {
      return null;
    }

    const itinerary = itineraryRows[0];
    const items = await this.dataSource.query(`
      SELECT id AS item_id, day_number, sequence_number, item_type, title, description
      FROM product_itinerary_items
      WHERE itinerary_id = $1
      ORDER BY day_number ASC, sequence_number ASC
    `, [itinerary.id]);

    const maxDay = items.reduce((max: number, it: any) => Math.max(max, it.day_number), 0);

    return {
      itineraryId: itinerary.id,
      productId,
      title: `${product.name} Itinerary`,
      daysCount: maxDay || (product as any).durationDays || 0,
      items: items.map((it: any) => ({
        itemId: it.item_id,
        dayNumber: it.day_number,
        sequenceNumber: it.sequence_number,
        itemType: it.item_type,
        title: it.title,
        description: it.description,
      })),
    };
  }

  async getLocations(productId: string) {
    await this.findById(productId);
    const rows = await this.dataSource.query(`
      SELECT
        pl.id AS location_id,
        pl.area_id,
        pl.area_name AS city,
        country.name AS country,
        country.code AS country_code,
        cont.name AS continent,
        pl.lat,
        pl.lng,
        pl.address,
        pl.sort_order
      FROM product_locations pl
      LEFT JOIN areas city ON city.id = pl.area_id
      LEFT JOIN areas country ON country.id = city.parent_id
      LEFT JOIN areas cont ON cont.id = country.parent_id
      WHERE pl.product_id = $1
      ORDER BY pl.sort_order ASC
    `, [productId]);

    return rows.map((r: any) => ({
      locationId: r.location_id,
      areaId: r.area_id,
      city: r.city,
      country: r.country,
      countryCode: r.country_code,
      continent: r.continent,
      lat: r.lat ? parseFloat(r.lat) : null,
      lng: r.lng ? parseFloat(r.lng) : null,
      address: r.address,
      sortOrder: r.sort_order,
    }));
  }

  async getVariants(productId: string) {
    await this.findById(productId);
    const rows = await this.dataSource.query(`
      SELECT
        v.id AS variant_id,
        v.code,
        v.name,
        v.slug,
        v.variant_type,
        v.listing_status,
        COALESCE(v.duration_days, pj.duration_days) AS duration_days,
        COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
        COALESCE(
          (
            SELECT MIN(ptp.selling_price)
            FROM product_trips pt
            INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
            WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
          ),
          0
        ) AS starting_price,
        (
          SELECT COUNT(*)
          FROM product_trips pt
          WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
        ) AS active_trips_count
      FROM product_variants v
      LEFT JOIN product_journeys pj ON pj.product_id = v.product_id AND pj.nationality_scope = 'ALL'
      WHERE v.product_id = $1 AND v.deleted_at IS NULL
      ORDER BY v.created_at ASC
    `, [productId]);

    return rows.map((r: any) => ({
      variantId: r.variant_id,
      code: r.code,
      name: r.name,
      slug: r.slug,
      variantType: r.variant_type,
      listingStatus: r.listing_status,
      durationDays: r.duration_days,
      durationNights: r.duration_nights,
      startingPrice: parseFloat(r.starting_price) || 0,
      activeTripsCount: parseInt(r.active_trips_count, 10) || 0,
    }));
  }

  async getSupplementaries(productId: string) {
    await this.findById(productId);
    const rows = await this.dataSource.query(`
      SELECT id, category, content, sort_order
      FROM product_supplementaries
      WHERE product_id = $1
      ORDER BY sort_order ASC
    `, [productId]);

    const titleMap: Record<string, string> = {
      INCLUDED: 'Paket Termasuk',
      EXCLUDED: 'Paket Tidak Termasuk',
      IMPORTANT_INFO: 'Informasi Penting',
      NOTE: 'Catatan Perjalanan',
    };

    return rows.map((r: any) => ({
      id: r.id,
      category: r.category,
      title: titleMap[r.category] || r.category,
      content: r.content,
    }));
  }

  async getSeo(productId: string) {
    await this.findById(productId);
    const rows = await this.dataSource.query(`
      SELECT
        id, target_type, target_id, meta_title, meta_description, canonical_url,
        og_title, og_description, og_image_url, no_index, no_follow, structured_data
      FROM seo_metadata
      WHERE target_type = 'PRODUCT' AND target_id = $1
      LIMIT 1
    `, [productId]);

    if (!rows.length) return null;
    const r = rows[0];
    return {
      id: r.id,
      targetType: r.target_type,
      targetId: r.target_id,
      metaTitle: r.meta_title,
      metaDescription: r.meta_description,
      canonicalUrl: r.canonical_url,
      ogTitle: r.og_title,
      ogDescription: r.og_description,
      ogImageUrl: r.og_image_url,
      noIndex: r.no_index,
      noFollow: r.no_follow,
      structuredData: r.structured_data,
    };
  }
}
```
