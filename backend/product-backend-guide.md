# Product Domain — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Implementation guide for the Product Domain backend service. Covers `ProductModule`, 2-tier Category binding, split sub-resource controllers (`/media`, `/locations`, `/variants`, `/supplementaries`, `/seo`), Variant default master itinerary & Trip override management, Variant Add-ons service, and Age-band trip pricing with itemized components.
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
│   ├── category.controller.ts            # 2-tier Category tree & CRUD
│   ├── variant-itinerary.controller.ts   # Variant master itinerary default
│   ├── trip-itinerary.controller.ts      # Trip itinerary override & resolution
│   ├── variant-addon.controller.ts       # Variant add-on management
│   └── trip-pricing.controller.ts        # Age-band pricing & quota
├── services/
│   ├── product.service.ts                # Master product lifecycle & transactions
│   ├── category.service.ts               # Category hierarchy management
│   ├── itinerary.service.ts              # Variant default & Trip override itineraries
│   ├── addon.service.ts                  # Optional extra add-ons
│   ├── pricing.service.ts                # Age bands & all-inclusive pricing
│   ├── product-location.service.ts       # 4-tier Area destination markers
│   └── product-supplementary.service.ts  # Inclusions, exclusions, terms
└── entities/                             # TypeORM schemas
    ├── product.entity.ts
    ├── product-category.entity.ts
    ├── product-journey.entity.ts
    ├── product-itinerary.entity.ts
    ├── product-addon.entity.ts
    ├── product-trip-pricing.entity.ts
    └── product-pricing-component.entity.ts
```

### Module Definition

```typescript
// product.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ProductController } from './controllers/product.controller';
import { CategoryController } from './controllers/category.controller';
import { VariantItineraryController } from './controllers/variant-itinerary.controller';
import { TripItineraryController } from './controllers/trip-itinerary.controller';
import { VariantAddonController } from './controllers/variant-addon.controller';
import { TripPricingController } from './controllers/trip-pricing.controller';

import { ProductService } from './services/product.service';
import { CategoryService } from './services/category.service';
import { ItineraryService } from './services/itinerary.service';
import { AddonService } from './services/addon.service';
import { PricingService } from './services/pricing.service';
import { ProductLocationService } from './services/product-location.service';
import { ProductSupplementaryService } from './services/product-supplementary.service';

import { Product } from './entities/product.entity';
import { ProductCategory } from './entities/product-category.entity';
import { ProductJourney } from './entities/product-journey.entity';
import { ProductTripPricing } from './entities/product-trip-pricing.entity';
import { ProductPricingComponent } from './entities/product-pricing-component.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([
      Product,
      ProductCategory,
      ProductJourney,
      ProductTripPricing,
      ProductPricingComponent,
    ]),
  ],
  controllers: [
    ProductController,
    CategoryController,
    VariantItineraryController,
    TripItineraryController,
    VariantAddonController,
    TripPricingController,
  ],
  providers: [
    ProductService,
    CategoryService,
    ItineraryService,
    AddonService,
    PricingService,
    ProductLocationService,
    ProductSupplementaryService,
  ],
  exports: [ProductService, CategoryService, ItineraryService, PricingService],
})
export class ProductModule {}
```

---

## 🎮 Controller Implementation

The base product controller handles headline catalog information and split sub-resources:

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

Creating a Product atomically links its child category (trigger resolves parent category) and stores its default journey duration:

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
   * Inserts the master product record with category link and its baseline journey duration.
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
        categoryId: dto.categoryId,
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
      .leftJoinAndSelect('p.category', 'cat')
      .leftJoinAndSelect('p.parentCategory', 'pcat')
      .where('p.deletedAt IS NULL');

    if (dto.listingStatus) {
      qb.andWhere('p.listingStatus = :status', { status: dto.listingStatus });
    }
    if (dto.productType) {
      qb.andWhere('p.productType = :type', { type: dto.productType });
    }
    if (dto.categoryId) {
      qb.andWhere('p.categoryId = :catId', { catId: dto.categoryId });
    }
    if (dto.categorySlug) {
      qb.andWhere('cat.slug = :catSlug', { catSlug: dto.categorySlug });
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
      relations: ['journeys', 'category', 'parentCategory'],
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

  async getLocations(productId: string) {
    await this.findById(productId);
    const rows = await this.dataSource.query(`
      SELECT
        pl.id AS location_id,
        pl.area_id,
        poi.name AS poi,
        country.name AS country,
        country.code AS country_code,
        sub.name AS sub_continent,
        cont.name AS continent,
        pl.lat,
        pl.lng,
        pl.address,
        pl.sort_order
      FROM product_locations pl
      LEFT JOIN areas poi ON poi.id = pl.area_id
      LEFT JOIN areas country ON country.id = poi.parent_id
      LEFT JOIN areas sub ON sub.id = country.parent_id
      LEFT JOIN areas cont ON cont.id = sub.parent_id
      WHERE pl.product_id = $1
      ORDER BY pl.sort_order ASC
    `, [productId]);

    return rows.map((r: any) => ({
      locationId: r.location_id,
      areaId: r.area_id,
      poi: r.poi,
      country: r.country,
      countryCode: r.country_code,
      subContinent: r.sub_continent,
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
            WHERE pt.variant_id = v.id
              AND pt.status = 'ACTIVE'
              AND pt.start_date >= CURRENT_DATE
              AND ptp.age_band = 'ADULT'
          ),
          0
        ) AS starting_price,
        (
          SELECT COUNT(*)
          FROM product_trips pt
          WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
        ) AS active_trips_count
      FROM product_variants v
      LEFT JOIN product_journeys pj ON pj.product_id = v.product_id
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
}
```

---

## 5. Trip Pricing & Itemized Breakdown Components Service

Trip pricings model age-band tiers (`ADULT`, `INFANT`) and itemized cost breakdown components (Flight, Hotel, Coach, Visa, Attractions).

### 5.1 TypeORM Entities

```typescript
// entities/product-trip-pricing.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, OneToMany, JoinColumn, CreateDateColumn, UpdateDateColumn } from 'typeorm';
import { ProductPricingComponent } from './product-pricing-component.entity';

@Entity('product_trip_pricings')
export class ProductTripPricing {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'trip_id', type: 'uuid' })
  tripId: string;

  @Column({ name: 'age_band', length: 30 })
  ageBand: 'ADULT' | 'INFANT';

  @Column({ name: 'age_min', type: 'int', nullable: true })
  ageMin: number;

  @Column({ name: 'age_max', type: 'int', nullable: true })
  ageMax: number;

  @Column({ name: 'consumes_quota', type: 'boolean', default: true })
  consumesQuota: boolean;

  @Column({ name: 'base_price', type: 'decimal', precision: 15, scale: 2 })
  basePrice: number;

  @Column({ name: 'selling_price', type: 'decimal', precision: 15, scale: 2 })
  sellingPrice: number;

  @Column({ length: 10, default: 'IDR' })
  currency: string;

  @OneToMany(() => ProductPricingComponent, (c) => c.pricing, { cascade: true })
  components: ProductPricingComponent[];

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

```typescript
// entities/product-pricing-component.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, JoinColumn, CreateDateColumn, UpdateDateColumn } from 'typeorm';
import { ProductTripPricing } from './product-trip-pricing.entity';

@Entity('product_pricing_components')
export class ProductPricingComponent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'pricing_id', type: 'uuid' })
  pricingId: string;

  @ManyToOne(() => ProductTripPricing, (pricing) => pricing.components, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'pricing_id' })
  pricing: ProductTripPricing;

  @Column({ length: 150 })
  name: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ type: 'decimal', precision: 15, scale: 2, nullable: true })
  amount: number;

  @Column({ name: 'is_included', type: 'boolean', default: true })
  isIncluded: boolean;

  @Column({ name: 'sort_order', type: 'int', default: 0 })
  sortOrder: number;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date;
}
```

### 5.2 Pricing Service Implementation

```typescript
// services/pricing.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { ProductTripPricing } from '../entities/product-trip-pricing.entity';
import { ProductPricingComponent } from '../entities/product-pricing-component.entity';

export interface CreatePricingComponentDto {
  name: string;
  description?: string;
  amount?: number;
  isIncluded?: boolean;
  sortOrder?: number;
}

export interface UpsertTripPricingDto {
  ageBand: 'ADULT' | 'INFANT';
  ageMin?: number;
  ageMax?: number;
  consumesQuota?: boolean;
  basePrice: number;
  sellingPrice: number;
  currency?: string;
  components?: CreatePricingComponentDto[];
}

@Injectable()
export class PricingService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * List all pricing tiers with their itemized components for a trip
   */
  async getTripPricings(tripId: string): Promise<ProductTripPricing[]> {
    const pricingRepo = this.dataSource.getRepository(ProductTripPricing);
    return pricingRepo.find({
      where: { tripId },
      relations: ['components'],
      order: { ageBand: 'ASC', components: { sortOrder: 'ASC' } },
    });
  }

  /**
   * Upsert a pricing tier and synchronize its itemized components within a transaction
   */
  async upsertTripPricing(tripId: string, dto: UpsertTripPricingDto): Promise<ProductTripPricing> {
    return this.dataSource.transaction(async (manager) => {
      const pricingRepo = manager.getRepository(ProductTripPricing);
      const componentRepo = manager.getRepository(ProductPricingComponent);

      let pricing = await pricingRepo.findOne({
        where: { tripId, ageBand: dto.ageBand },
      });

      if (!pricing) {
        pricing = pricingRepo.create({
          tripId,
          ageBand: dto.ageBand,
          ageMin: dto.ageMin,
          ageMax: dto.ageMax,
          consumesQuota: dto.consumesQuota ?? (dto.ageBand === 'ADULT'),
          basePrice: dto.basePrice,
          sellingPrice: dto.sellingPrice,
          currency: dto.currency || 'IDR',
        });
      } else {
        pricing.basePrice = dto.basePrice;
        pricing.sellingPrice = dto.sellingPrice;
        if (dto.ageMin !== undefined) pricing.ageMin = dto.ageMin;
        if (dto.ageMax !== undefined) pricing.ageMax = dto.ageMax;
        if (dto.consumesQuota !== undefined) pricing.consumesQuota = dto.consumesQuota;
        if (dto.currency) pricing.currency = dto.currency;
      }

      const savedPricing = await pricingRepo.save(pricing);

      // Synchronize itemized components if provided
      if (dto.components) {
        await componentRepo.delete({ pricingId: savedPricing.id });
        if (dto.components.length > 0) {
          const components = dto.components.map((c, idx) =>
            componentRepo.create({
              pricingId: savedPricing.id,
              name: c.name,
              description: c.description,
              amount: c.amount,
              isIncluded: c.isIncluded ?? true,
              sortOrder: c.sortOrder ?? idx,
            }),
          );
          await componentRepo.save(components);
        }
      }

      return pricingRepo.findOne({
        where: { id: savedPricing.id },
        relations: ['components'],
      });
    });
  }
}
```
