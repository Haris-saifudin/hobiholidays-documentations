# Product Hierarchy — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend implementation guide for the 3-Level Product Hierarchy (`Product → Variant → Trip → Pricing`). Covers Variant catalog aggregation, duration inheritance resolution via SQL `COALESCE`, and booking transaction safety with Pessimistic Locking (`SELECT ... FOR UPDATE`).
>
> **Related Design Document:** [Product Hierarchy Technical Design](../technical/product-hierarchy-technical-design.md)
> **API Contract:** [Product Hierarchy Contracts](../contracts/product-hierarchy-contract.md)
> **Frontend Guide:** [Product Hierarchy Frontend Guide](../frontend/product-hierarchy-frontend-guide.md)

---

## 🏗️ Module Overview

The hierarchy module manages the relationship chain between:
1. **Products (L1):** Master brand context.
2. **Product Variants (L2):** Named editions (All Tours catalog cards).
3. **Product Trips (L3):** Departure calendar windows and passenger capacity.
4. **Product Trip Pricings (L3+):** Tiered pricing resolved by nationality scope.

```
src/modules/product-hierarchy/
├── product-hierarchy.module.ts
├── controllers/
│   ├── variant.controller.ts             # Public All Tours feed & PDP
│   └── trip-booking.controller.ts        # Departure selection & quota reservation
├── services/
│   ├── product-hierarchy.service.ts      # Catalog aggregation & inheritance
│   └── booking-transaction.service.ts    # Pessimistic lock quota management
└── dto/
    ├── list-variants.dto.ts
    └── reserve-trip-quota.dto.ts
```

---

## ⚙️ Variant Catalog Aggregation & Duration Inheritance

Variants inherit duration from their parent Product's baseline `product_journeys` row unless an explicit override is set on `product_variants`. The backend service resolves this dynamically using `COALESCE`:

```typescript
// services/product-hierarchy.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { ListVariantsDto } from '../dto/list-variants.dto';

@Injectable()
export class ProductHierarchyService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Retrieves the All Tours listing feed with resolved durations,
   * starting prices, and hero cover images.
   */
  async getCatalogFeed(dto: ListVariantsDto) {
    const page = dto.page || 1;
    const limit = dto.limit || 12;
    const offset = (page - 1) * limit;

    const query = `
      SELECT
        v.id,
        v.code,
        v.name,
        v.slug,
        v.variant_type,
        v.listing_status,
        -- Duration inheritance resolution:
        COALESCE(v.duration_days, pj.duration_days) AS duration_days,
        COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
        -- Parent master brand info:
        p.id AS product_id,
        p.name AS product_name,
        p.slug AS product_slug,
        -- Starting price aggregation:
        COALESCE(
          (
            SELECT MIN(ptp.selling_price)
            FROM product_trips pt
            INNER JOIN product_trip_pricings ptp ON ptp.trip_id = pt.id
            WHERE pt.variant_id = v.id
              AND pt.status = 'ACTIVE'
              AND pt.start_date >= CURRENT_DATE
              AND (ptp.nationality_scope = $1 OR ptp.nationality_scope = 'ALL')
          ),
          0
        ) AS starting_price,
        -- Hero cover image:
        (
          SELECT m.url
          FROM product_media_usages pmu
          INNER JOIN product_media m ON m.id = pmu.media_id
          WHERE (pmu.target_type = 'VARIANT' AND pmu.target_id = v.id AND pmu.usage_context = 'COVER')
             OR (pmu.target_type = 'PRODUCT' AND pmu.target_id = p.id AND pmu.usage_context = 'COVER')
          ORDER BY (CASE WHEN pmu.target_type = 'VARIANT' THEN 1 ELSE 2 END) ASC
          LIMIT 1
        ) AS cover_url,
        -- Next upcoming departure date:
        (
          SELECT MIN(pt.start_date)
          FROM product_trips pt
          WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
        ) AS next_departure_date,
        -- Total active upcoming departures count:
        (
          SELECT COUNT(*)
          FROM product_trips pt
          WHERE pt.variant_id = v.id AND pt.status = 'ACTIVE' AND pt.start_date >= CURRENT_DATE
        ) AS total_active_departures,
        -- Destination stop markers:
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'city', a_city.name,
              'country', a_country.name,
              'continent', a_cont.name
            ) ORDER BY pl.sort_order ASC)
            FROM product_locations pl
            INNER JOIN areas a_city ON a_city.id = pl.area_id
            INNER JOIN areas a_country ON a_country.id = a_city.parent_id
            INNER JOIN areas a_cont ON a_cont.id = a_country.parent_id
            WHERE pl.product_id = p.id
          ),
          '[]'::json
        ) AS destinations
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id AND pj.nationality_scope = 'ALL'
      WHERE v.listing_status = 'ACTIVE'
        AND p.listing_status = 'ACTIVE'
        AND p.deleted_at IS NULL
      ORDER BY v.created_at DESC
      LIMIT $2 OFFSET $3;
    `;

    const scope = dto.nationalityScope || 'ALL';
    const rows = await this.dataSource.query(query, [scope, limit, offset]);

    const countResult = await this.dataSource.query(`
      SELECT COUNT(*) AS total
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      WHERE v.listing_status = 'ACTIVE' AND p.listing_status = 'ACTIVE' AND p.deleted_at IS NULL
    `);

    const totalItems = parseInt(countResult[0].total, 10);
    const totalPages = Math.ceil(totalItems / limit);

    const getBadge = (type: string) => {
      switch (type) {
        case 'SEASONAL': return '🌸 Seasonal';
        case 'THEMED': return '🌷 Special Edition';
        case 'PROMOTIONAL': return '🔥 Limited Offer';
        default: return null;
      }
    };

    const data = rows.map((r: any) => ({
      variantId: r.id,
      code: r.code,
      name: r.name,
      slug: r.slug,
      variantType: r.variant_type,
      badge: getBadge(r.variant_type),
      productId: r.product_id,
      productName: r.product_name,
      productSlug: r.product_slug,
      durationDays: r.duration_days,
      durationNights: r.duration_nights,
      coverUrl: r.cover_url || 'https://cdn.hobiholidays.com/defaults/cover.jpg',
      destinations: r.destinations || [],
      startingPrice: parseFloat(r.starting_price) || 0,
      currency: 'IDR',
      nextDepartureDate: r.next_departure_date || null,
      totalActiveDepartures: parseInt(r.total_active_departures, 10) || 0,
    }));

    return {
      statusCode: 200,
      message: 'All Tours catalog retrieved successfully',
      meta: {
        totalItems,
        itemCount: data.length,
        itemsPerPage: limit,
        totalPages,
        currentPage: page,
      },
      data,
    };
  }

  /**
   * Fetches full public details for a Variant Detail page (PDP).
   */
  async getVariantBySlug(slug: string, scope: string = 'ALL') {
    const variantQuery = `
      SELECT
        v.id, v.code, v.name, v.slug, v.variant_type, v.listing_status,
        COALESCE(v.duration_days, pj.duration_days) AS duration_days,
        COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
        p.id AS product_id, p.name AS product_name, p.slug AS product_slug,
        p.itinerary_pdf_url
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id AND pj.nationality_scope = 'ALL'
      WHERE v.slug = $1 AND v.listing_status = 'ACTIVE'
      LIMIT 1;
    `;

    const variants = await this.dataSource.query(variantQuery, [slug]);
    if (!variants.length) {
      throw new NotFoundException(`Variant '${slug}' not found or inactive`);
    }

    const variant = variants[0];

    // Fetch upcoming departures with available capacity
    const trips = await this.dataSource.query(`
      SELECT
        t.id, t.trip_code, t.start_date, t.end_date, t.status,
        t.min_quota, t.max_quota,
        (t.max_quota - COALESCE(b.booked_count, 0)) AS available_seats,
        ptp.selling_price, ptp.currency, ptp.nationality_scope
      FROM product_trips t
      LEFT JOIN (
        SELECT trip_id, SUM(pax_count) AS booked_count
        FROM product_trip_bookings
        WHERE booking_status IN ('CONFIRMED', 'PAID')
        GROUP BY trip_id
      ) b ON b.trip_id = t.id
      LEFT JOIN product_trip_pricings ptp ON ptp.trip_id = t.id AND (ptp.nationality_scope = $2 OR ptp.nationality_scope = 'ALL')
      WHERE t.variant_id = $1
        AND t.status = 'ACTIVE'
        AND t.start_date >= CURRENT_DATE
      ORDER BY t.start_date ASC;
    `, [variant.id, scope]);

    return {
      variantId: variant.id,
      code: variant.code,
      name: variant.name,
      slug: variant.slug,
      variantType: variant.variant_type,
      durationDays: variant.duration_days,
      durationNights: variant.duration_nights,
      productId: variant.product_id,
      productName: variant.product_name,
      productSlug: variant.product_slug,
      itineraryPdfUrl: variant.itinerary_pdf_url,
      trips: trips.map((t: any) => ({
        tripId: t.id,
        tripCode: t.trip_code,
        startDate: t.start_date,
        endDate: t.end_date,
        status: t.status,
        minQuota: t.min_quota,
        maxQuota: t.max_quota,
        availableSeats: parseInt(t.available_seats, 10),
        sellingPrice: parseFloat(t.selling_price) || 0,
        currency: t.currency || 'IDR',
        nationalityScope: t.nationality_scope,
      })),
    };
  }
}
```

---

## 🔒 Quota Concurrency: Pessimistic Locking

To guarantee that concurrent travelers cannot over-book a departure window beyond `product_trips.max_quota`, the booking service acquires a row-level exclusive lock using **`SELECT ... FOR UPDATE`**:

```typescript
// services/booking-transaction.service.ts
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';

export interface ReserveSeatDto {
  tripId: string;
  paxCount: number;
  customerId: string;
}

@Injectable()
export class BookingTransactionService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Executes seat reservation within a strict ACID transaction using
   * Pessimistic Exclusive Locking (SELECT ... FOR UPDATE).
   */
  async reserveSeats(dto: ReserveSeatDto) {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Lock the specific product_trips row exclusively
      const tripRows = await queryRunner.query(
        `SELECT id, variant_id, status, max_quota, start_date
         FROM product_trips
         WHERE id = $1
         FOR UPDATE`,
        [dto.tripId],
      );

      if (!tripRows.length) {
        throw new NotFoundException(`Trip '${dto.tripId}' not found`);
      }

      const trip = tripRows[0];

      if (trip.status !== 'ACTIVE') {
        throw new ConflictException(`Trip is not open for booking (status: ${trip.status})`);
      }

      // 2. Count confirmed seats already allocated
      const bookingSum = await queryRunner.query(
        `SELECT COALESCE(SUM(pax_count), 0) AS total_booked
         FROM product_trip_bookings
         WHERE trip_id = $1 AND booking_status IN ('PENDING_PAYMENT', 'CONFIRMED', 'PAID')`,
        [dto.tripId],
      );

      const totalBooked = parseInt(bookingSum[0].total_booked, 10);
      const remainingQuota = trip.max_quota - totalBooked;

      if (remainingQuota < dto.paxCount) {
        throw new ConflictException(
          `Insufficient quota: requested ${dto.paxCount} seats, but only ${remainingQuota} seats remain`,
        );
      }

      // 3. Insert reservation record
      const insertResult = await queryRunner.query(
        `INSERT INTO product_trip_bookings
          (trip_id, customer_id, pax_count, booking_status, expires_at)
         VALUES ($1, $2, $3, 'PENDING_PAYMENT', NOW() + INTERVAL '15 minutes')
         RETURNING id`,
        [dto.tripId, dto.customerId, dto.paxCount],
      );

      // 4. Auto-update trip status to 'FULL' if quota reached 0
      if (remainingQuota - dto.paxCount === 0) {
        await queryRunner.query(
          `UPDATE product_trips SET status = 'FULL' WHERE id = $1`,
          [dto.tripId],
        );
      }

      await queryRunner.commitTransaction();

      return {
        bookingId: insertResult[0].id,
        tripId: dto.tripId,
        reservedPax: dto.paxCount,
        expiresInMinutes: 15,
      };
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```
