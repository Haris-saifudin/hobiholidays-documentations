# Product Hierarchy — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend implementation guide for the 3-Level Product Hierarchy (`Product → Variant → Trip → Pricing`). Covers Variant catalog aggregation with category badges, duration inheritance resolution via SQL `COALESCE`, default master itinerary with trip override resolution (`trip.itinerary ?? variant.itinerary`), and booking transaction safety with Pessimistic Locking (`SELECT ... FOR UPDATE`) enforcing `consumes_quota` seat allocation rules.
>
> **Related Design Document:** [Product Hierarchy Technical Design](../technical/product-hierarchy-technical-design.md)  
> **API Contract:** [Product Hierarchy Contracts](../contracts/product-hierarchy-contract.md)  
> **Frontend Guide:** [Product Hierarchy Frontend Guide](../frontend/product-hierarchy-frontend-guide.md)

---

## 🏗️ Module Overview

The hierarchy module manages the relationship chain between:
1. **Products (L1):** Master brand umbrella + 2-tier Category taxonomy.
2. **Product Variants (L2):** Named editions (All Tours catalog cards), default master itinerary, and base add-ons.
3. **Product Trips (L3):** Departure calendar windows, quota, and optional itinerary overrides.
4. **Product Trip Pricings (L3+):** All-inclusive package pricing by age band (`ADULT`, `INFANT`) with dynamic `consumes_quota`.

```
src/modules/product-hierarchy/
├── product-hierarchy.module.ts
├── controllers/
│   ├── variant.controller.ts             # Public All Tours feed & PDP
│   ├── itinerary-resolution.controller.ts# Trip vs Variant itinerary fallback
│   └── trip-booking.controller.ts        # Departure selection & quota reservation
├── services/
│   ├── product-hierarchy.service.ts      # Catalog aggregation & inheritance
│   ├── itinerary-resolution.service.ts   # trip.itinerary ?? variant.itinerary
│   └── booking-transaction.service.ts    # Pessimistic lock quota management
└── dto/
    ├── list-variants.dto.ts
    └── reserve-trip-quota.dto.ts
```

---

## ⚙️ Variant Catalog Aggregation & Duration Inheritance

Variants inherit duration from their parent Product's baseline `product_journeys` row unless an explicit override is set on `product_variants`. The backend service resolves this dynamically using `COALESCE`, joining 2-tier categories and 4-tier geography:

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
   * category metadata, starting prices (ADULT), and cover images.
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
        -- Parent master brand info & Category taxonomy:
        p.id AS product_id,
        p.name AS product_name,
        p.slug AS product_slug,
        cat.id AS category_id,
        cat.name AS category_name,
        cat.slug AS category_slug,
        pcat.id AS parent_category_id,
        pcat.name AS parent_category_name,
        pcat.slug AS parent_category_slug,
        -- Starting price aggregation (Lowest ADULT selling_price):
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
        -- 4-tier Destination stop markers:
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'poi', a_poi.name,
              'country', a_country.name,
              'subContinent', a_sub.name,
              'continent', a_cont.name
            ) ORDER BY pl.sort_order ASC)
            FROM product_locations pl
            INNER JOIN areas a_poi ON a_poi.id = pl.area_id
            LEFT JOIN areas a_country ON a_country.id = a_poi.parent_id
            LEFT JOIN areas a_sub ON a_sub.id = a_country.parent_id
            LEFT JOIN areas a_cont ON a_cont.id = a_sub.parent_id
            WHERE pl.product_id = p.id
          ),
          '[]'::json
        ) AS destinations,
        -- Custom Promotional Badges (M:N via product_variant_badges):
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'id', pb.id,
              'code', pb.code,
              'label', pb.label,
              'backgroundColor', pb.background_color,
              'textColor', pb.text_color,
              'iconUrl', pb.icon_url
            ) ORDER BY pb.created_at ASC)
            FROM product_variant_badges pvb
            INNER JOIN product_badges pb ON pb.id = pvb.badge_id
            WHERE pvb.variant_id = v.id AND pb.is_active = TRUE
          ),
          '[]'::json
        ) AS badges
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      LEFT JOIN product_categories cat ON cat.id = p.category_id
      LEFT JOIN product_categories pcat ON pcat.id = p.parent_category_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id
      WHERE v.listing_status = 'ACTIVE'
        AND p.listing_status = 'ACTIVE'
        AND p.deleted_at IS NULL
      ORDER BY v.created_at DESC
      LIMIT $1 OFFSET $2;
    `;

    const rows = await this.dataSource.query(query, [limit, offset]);

    const countResult = await this.dataSource.query(`
      SELECT COUNT(*) AS total
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      WHERE v.listing_status = 'ACTIVE' AND p.listing_status = 'ACTIVE' AND p.deleted_at IS NULL
    `);

    const totalItems = parseInt(countResult[0].total, 10);
    const totalPages = Math.ceil(totalItems / limit);

    const data = rows.map((r: any) => ({
      variantId: r.id,
      code: r.code,
      name: r.name,
      slug: r.slug,
      variantType: r.variant_type,
      badges: r.badges || [],
      productId: r.product_id,
      productName: r.product_name,
      productSlug: r.product_slug,
      category: r.category_id ? { id: r.category_id, name: r.category_name, slug: r.category_slug } : null,
      parentCategory: r.parent_category_id ? { id: r.parent_category_id, name: r.parent_category_name, slug: r.parent_category_slug } : null,
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
   * Note: Itinerary PDF brochures are compiled externally by ATW.
   * Hobiholidays does not generate PDFs internally; it resolves COALESCE(v.itinerary_pdf_url, p.itinerary_pdf_url).
   */
  async getVariantBySlug(slug: string) {
    const variantQuery = `
      SELECT
        v.id, v.code, v.name, v.slug, v.variant_type, v.listing_status,
        COALESCE(v.duration_days, pj.duration_days) AS duration_days,
        COALESCE(v.duration_nights, pj.duration_nights) AS duration_nights,
        COALESCE(v.itinerary_pdf_url, p.itinerary_pdf_url) AS itinerary_pdf_url,
        p.id AS product_id, p.name AS product_name, p.slug AS product_slug,
        cat.id AS category_id, cat.name AS category_name, cat.slug AS category_slug,
        pcat.id AS parent_category_id, pcat.name AS parent_category_name, pcat.slug AS parent_category_slug
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
      LEFT JOIN product_categories cat ON cat.id = p.category_id
      LEFT JOIN product_categories pcat ON pcat.id = p.parent_category_id
      LEFT JOIN product_journeys pj ON pj.product_id = p.id
      WHERE v.slug = $1 AND v.listing_status = 'ACTIVE'
      LIMIT 1;
    `;

    const variants = await this.dataSource.query(variantQuery, [slug]);
    if (!variants.length) {
      throw new NotFoundException(`Variant '${slug}' not found or inactive`);
    }

    const variant = variants[0];

    // Fetch default master itinerary for this variant
    const defaultItinerary = await this.dataSource.query(`
      SELECT id, title, days_count
      FROM product_itineraries
      WHERE variant_id = $1 AND trip_id IS NULL AND is_active = TRUE
      LIMIT 1;
    `, [variant.id]);

    // Fetch optional add-ons configured for this variant
    const addons = await this.dataSource.query(`
      SELECT id, code, name, description, addon_type, charge_type, price, currency, applicable_age_band, is_mandatory, max_quantity
      FROM product_addons
      WHERE variant_id = $1 AND is_active = TRUE
      ORDER BY name ASC;
    `, [variant.id]);

    // Fetch upcoming departures with available capacity and age-band pricings
    const trips = await this.dataSource.query(`
      SELECT
        t.id, t.trip_code, t.start_date, t.end_date, t.status,
        t.min_quota, t.max_quota,
        (t.max_quota - COALESCE(b.booked_count, 0)) AS available_seats,
        EXISTS(SELECT 1 FROM product_itineraries pi WHERE pi.trip_id = t.id AND pi.is_active = TRUE) AS has_trip_override
      FROM product_trips t
      LEFT JOIN (
        SELECT ptb.trip_id, SUM(CASE WHEN ptp.consumes_quota THEN 1 ELSE 0 END) AS booked_count
        FROM product_trip_bookings ptb
        INNER JOIN product_trip_pricings ptp ON ptp.id = ptb.pricing_id
        WHERE ptb.booking_status IN ('CONFIRMED', 'PAID')
        GROUP BY ptb.trip_id
      ) b ON b.trip_id = t.id
      WHERE t.variant_id = $1
        AND t.status = 'ACTIVE'
        AND t.start_date >= CURRENT_DATE
      ORDER BY t.start_date ASC;
    `, [variant.id]);

    return {
      variantId: variant.id,
      code: variant.code,
      name: variant.name,
      slug: variant.slug,
      variantType: variant.variant_type,
      durationDays: variant.duration_days,
      durationNights: variant.duration_nights,
      itineraryPdfUrl: variant.itinerary_pdf_url,
      product: {
        id: variant.product_id,
        name: variant.product_name,
        slug: variant.product_slug,
        category: variant.category_id ? { id: variant.category_id, name: variant.category_name, slug: variant.category_slug } : null,
        parentCategory: variant.parent_category_id ? { id: variant.parent_category_id, name: variant.parent_category_name, slug: variant.parent_category_slug } : null,
      },
      itinerary: defaultItinerary[0] || null,
      addons,
      trips,
    };
  }
}
```

---

## 🗺️ Itinerary Fallback Resolution: `trip.itinerary ?? variant.itinerary`

When rendering the day-by-day itinerary on a dated departure, the backend checks for a trip-specific override first; if null, it falls back to the variant default:

```typescript
// services/itinerary-resolution.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class ItineraryResolutionService {
  constructor(private readonly dataSource: DataSource) {}

  async resolveEffectiveItinerary(tripId: string) {
    const tripRows = await this.dataSource.query(
      `SELECT id, variant_id FROM product_trips WHERE id = $1`,
      [tripId],
    );
    if (!tripRows.length) throw new NotFoundException(`Trip '${tripId}' not found`);

    const { variant_id: variantId } = tripRows[0];

    // Check for Trip-level override itinerary first
    const tripItinerary = await this.dataSource.query(`
      SELECT id, title, days_count, TRUE AS is_override
      FROM product_itineraries
      WHERE trip_id = $1 AND is_active = TRUE
      LIMIT 1;
    `, [tripId]);

    if (tripItinerary.length) {
      const items = await this.fetchItineraryItems(tripItinerary[0].id);
      return { ...tripItinerary[0], items };
    }

    // Fallback to Variant-level default itinerary
    const variantItinerary = await this.dataSource.query(`
      SELECT id, title, days_count, FALSE AS is_override
      FROM product_itineraries
      WHERE variant_id = $1 AND trip_id IS NULL AND is_active = TRUE
      LIMIT 1;
    `, [variantId]);

    if (variantItinerary.length) {
      const items = await this.fetchItineraryItems(variantItinerary[0].id);
      return { ...variantItinerary[0], items };
    }

    return null;
  }

  private async fetchItineraryItems(itineraryId: string) {
    return this.dataSource.query(`
      SELECT day_number, sequence_number, item_type, title, description, accommodation, location_name
      FROM product_itinerary_items
      WHERE itinerary_id = $1
      ORDER BY day_number ASC, sequence_number ASC;
    `, [itineraryId]);
  }
}
```

---

## 🔒 Quota Concurrency: Pessimistic Locking & `consumes_quota` Rule

To guarantee that concurrent bookings never over-allocate seats, the booking transaction locks the trip row exclusively (`SELECT ... FOR UPDATE`). Only passengers with `consumes_quota = TRUE` consume seat capacity (infants are configured via `consumes_quota`: `FALSE` for lap infants, or `TRUE` if an aircraft/bus seat or cot is allocated):

```typescript
// services/booking-transaction.service.ts
import { Injectable, ConflictException, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';

export interface PassengerBookingItem {
  passengerName: string;
  ageBand: 'ADULT' | 'INFANT';
}

export interface ReserveSeatDto {
  tripId: string;
  customerId: string;
  passengers: PassengerBookingItem[];
}

@Injectable()
export class BookingTransactionService {
  constructor(private readonly dataSource: DataSource) {}

  async reserveSeats(dto: ReserveSeatDto) {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Lock the trip row exclusively
      const tripRows = await queryRunner.query(
        `SELECT id, variant_id, status, max_quota, start_date
         FROM product_trips
         WHERE id = $1
         FOR UPDATE`,
        [dto.tripId],
      );

      if (!tripRows.length) throw new NotFoundException(`Trip '${dto.tripId}' not found`);
      const trip = tripRows[0];

      if (trip.status !== 'ACTIVE') {
        throw new ConflictException(`Trip is not open for booking (status: ${trip.status})`);
      }

      // 2. Fetch pricing rules for age bands to identify consumes_quota
      const pricingTiers = await queryRunner.query(
        `SELECT id, age_band, consumes_quota
         FROM product_trip_pricings
         WHERE trip_id = $1`,
        [dto.tripId],
      );
      const quotaRuleMap = new Map<string, boolean>();
      pricingTiers.forEach((p: any) => quotaRuleMap.set(p.age_band, p.consumes_quota));

      // Calculate seats required (only count passengers where consumes_quota = TRUE)
      const seatsRequired = dto.passengers.filter(
        (p) => quotaRuleMap.get(p.ageBand) !== false,
      ).length;

      // 3. Count confirmed seats already allocated
      const bookingSum = await queryRunner.query(
        `SELECT COALESCE(SUM(CASE WHEN ptp.consumes_quota THEN 1 ELSE 0 END), 0) AS total_booked
         FROM product_trip_bookings ptb
         INNER JOIN product_trip_pricings ptp ON ptp.id = ptb.pricing_id
         WHERE ptb.trip_id = $1 AND ptb.booking_status IN ('PENDING_PAYMENT', 'CONFIRMED', 'PAID')`,
        [dto.tripId],
      );

      const totalBooked = parseInt(bookingSum[0].total_booked, 10);
      const remainingQuota = trip.max_quota - totalBooked;

      if (remainingQuota < seatsRequired) {
        throw new ConflictException(
          `Insufficient quota: requested ${seatsRequired} seats, but only ${remainingQuota} seats remain`,
        );
      }

      // 4. Insert booking records
      const bookingResult = await queryRunner.query(
        `INSERT INTO product_trip_bookings
          (trip_id, customer_id, pax_count, booking_status, expires_at)
         VALUES ($1, $2, $3, 'PENDING_PAYMENT', NOW() + INTERVAL '15 minutes')
         RETURNING id`,
        [dto.tripId, dto.customerId, dto.passengers.length],
      );

      // Auto-update trip status to 'FULL' if quota reached
      if (remainingQuota - seatsRequired === 0) {
        await queryRunner.query(
          `UPDATE product_trips SET status = 'FULL' WHERE id = $1`,
          [dto.tripId],
        );
      }

      await queryRunner.commitTransaction();

      return {
        bookingId: bookingResult[0].id,
        tripId: dto.tripId,
        seatsAllocated: seatsRequired,
        totalPassengers: dto.passengers.length,
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
