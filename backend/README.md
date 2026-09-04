# NestJS Backend Architecture & Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Comprehensive backend engineering guidelines, NestJS module layouts, controller and service patterns, database transaction lifecycles, streaming controllers, and database query implementations for the Hobiholidays platform.
>
> _Target Audience: Backend Engineers, NestJS Developers, and API Platform Engineers._

---

## 🗺️ Backend Guides Directory Map

| Guide Document | Domain Scope | Primary Backend Implementation Focus |
| :--- | :--- | :--- |
| **[Product Backend Guide](./product-backend-guide.md)** | Master Products & Catalog | `ProductModule`, `ProductService`, split sub-resource controller architecture (`/media`, `/itineraries`, `/locations`, `/variants`, `/supplementaries`, `/seo`), and transaction boundaries. |
| **[Product Hierarchy Backend Guide](./product-hierarchy-backend-guide.md)** | 3-Level Catalog & Booking | `ProductHierarchyService`, `COALESCE` duration inheritance query, pessimistic locking (`SELECT ... FOR UPDATE`) during booking transactions to prevent quota over-allocation. |
| **[Area Domain Backend Guide](./area-backend-guide.md)** | Geographic Domain | `AreaModule`, `AreaService`, recursive Common Table Expression (CTE) tree traversal queries, PostGIS boundary calculations, and in-memory reference caching. |
| **[Search & Filter Backend Guide](./product-search-filter-backend-guide.md)** | Discovery Engine | `SearchFilterService`, parameterized dynamic SQL query builder, trigram similarity matching (`pg_trgm`), and window function result counting (`COUNT(*) OVER()`). |
| **[Product Media Backend Guide](./product-media-backend-guide.md)** | Media Subsystem | `MediaModule`, Multer 25MB file interceptor, `BYTEA` streaming controller with HTTP cache headers, AWS SDK S3 presigned URL generator, and zero-downtime Phase 1 to Phase 2 migration script. |
| **[SEO Backend Guide](./seo-backend-guide.md)** | SEO & Structured Data | `SeoModule`, `SeoService`, polymorphic fallback resolver (Layer A custom DB vs Layer B formulaic dynamic fallback), and Schema.org JSON-LD generator helper. |

---

## 🏛️ Cross-Pillar References

- **Data Models & DDL:** [Technical Architecture](../technical/README.md) — Authoritative PostgreSQL DDL schemas, ERDs, triggers, and indexes.
- **API Contracts:** [REST API Contracts](../contracts/README.md) — Endpoints, NestJS `class-validator` DTOs, and response envelopes.
- **Frontend Integration:** [Next.js Frontend Guides](../frontend/README.md) — Server component data fetching, ISR revalidation, and UI states.

---

## 🏗️ NestJS Application Architecture Standards

### 1. Project Directory Layout
All backend modules adhere to standard NestJS modular architecture:

```
src/
├── common/
│   ├── decorators/          # Custom parameter & auth decorators
│   ├── dto/                 # Global DTOs (PaginationQueryDto, StandardMetaDto)
│   ├── filters/             # Global RFC 7807 HttpExceptionFilter
│   ├── interceptors/        # Logging, Transform, and Cache interceptors
│   └── pipes/               # ParseUUIDPipe, SanitizePipe
├── config/                  # Environment config validation (@nestjs/config)
├── database/                # DatabaseModule, TypeORM / Kysely connection pool
└── modules/
    ├── product/             # ProductModule (L1 Master Catalog)
    ├── product-hierarchy/   # ProductHierarchyModule (L2/L3 Variant, Trip, Pricing)
    ├── area/                # AreaModule (Continents, Countries, Cities)
    ├── search/              # SearchFilterModule (Search & Filter Query Engine)
    ├── media/               # MediaModule (Phase 1 BYTEA & Phase 2 S3/R2)
    └── seo/                 # SeoModule (Polymorphic Metadata & Fallbacks)
```

### 2. Global Validation Pipe Configuration
All incoming HTTP requests pass through the global validation pipe in `main.ts`:

```typescript
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { HttpExceptionFilter } from './common/filters/http-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api/v1');

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,               // Strip unwhitelisted properties
      forbidNonWhitelisted: true,    // Throw 400 if unknown props sent
      transform: true,               // Auto-transform payloads to DTO instances
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(3001);
}
bootstrap();
```

### 3. RFC 7807 Global Exception Filter
Ensures all errors emit the standard `ApiErrorResponse` structure:

```typescript
// common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const exceptionResponse =
      exception instanceof HttpException ? exception.getResponse() : null;

    let message: string | string[] = 'Internal server error';

    if (typeof exceptionResponse === 'object' && exceptionResponse !== null) {
      const respObj = exceptionResponse as any;
      message = respObj.message || message;
    } else if (typeof exceptionResponse === 'string') {
      message = exceptionResponse;
    }

    response.status(status).json({
      statusCode: status,
      message,
      error: HttpStatus[status] || 'Error',
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

### 4. Database Transaction Management
To maintain atomicity across multi-table operations (e.g. creating a Product + initial Journey row, or reserving Trip Quota), services execute operations within database transactions:

```typescript
import { DataSource } from 'typeorm';
import { Injectable } from '@nestjs/common';

@Injectable()
export class BaseTransactionalService {
  constructor(protected readonly dataSource: DataSource) {}

  protected async runInTransaction<T>(
    operation: (queryRunner: any) => Promise<T>,
  ): Promise<T> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();
    try {
      const result = await operation(queryRunner);
      await queryRunner.commitTransaction();
      return result;
    } catch (err) {
      await queryRunner.rollbackTransaction();
      throw err;
    } finally {
      await queryRunner.release();
    }
  }
}
```
