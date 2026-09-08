# Product Hierarchy — NestJS Backend Implementation Guide

> **Pillar 3: NestJS Backend Implementation**
> Backend implementation guide for the 3-Level Product Hierarchy (`Product → Variant → Trip → Pricing`). Covers Variant catalog aggregation with category badges, duration inheritance resolution via SQL `COALESCE`, default master itinerary with trip override resolution (`trip.itinerary ?? variant.itinerary`), and read-optimized departure schedule retrieval with nominal seat availability indicators (`availableSeats = max_quota - booked_seats`).
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
│   └── trip-availability.controller.ts   # Departure selection & capacity indicators
├── services/
│   ├── product-hierarchy.service.ts      # Catalog aggregation & inheritance
│   ├── itinerary-resolution.service.ts   # trip.itinerary ?? variant.itinerary
│   └── trip-availability.service.ts      # Nominal capacity & available seats aggregation
└── dto/
    ├── list-variants.dto.ts
    └── list-variant-trips.dto.ts
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
        -- Parent master brand info:
        p.id AS product_id,
        p.name AS product_name,
        p.slug AS product_slug,
        -- Multi-dimensional Category taxonomy (via product_category_assignments):
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'id', pc.id,
              'name', pc.name,
              'slug', pc.slug,
              'dimensionCode', cd.code,
              'dimensionName', cd.name
            ) ORDER BY cd.sort_order ASC, pc.sort_order ASC)
            FROM product_category_assignments pca
            INNER JOIN product_categories pc ON pc.id = pca.category_id
            INNER JOIN category_dimensions cd ON cd.id = pc.dimension_id
            WHERE pca.product_id = p.id AND pc.is_active = TRUE
          ),
          '[]'::json
        ) AS categories,
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
        -- Flexible Flat Destination stop markers (dynamic upward traversal):
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'continent', continent_area.name,
              'continentSlug', continent_area.slug,
              'subContinent', subcont_area.name,
              'subContinentSlug', subcont_area.slug,
              'country', country_area.name,
              'countrySlug', country_area.slug,
              'poi', CASE WHEN target_area.area_type_id = 4 THEN COALESCE(pl.area_name, target_area.name) ELSE NULL END,
              'poiSlug', CASE WHEN target_area.area_type_id = 4 THEN target_area.slug ELSE NULL END
            ) ORDER BY pl.sort_order ASC)
            FROM product_locations pl
            INNER JOIN areas target_area ON target_area.id = pl.area_id
            LEFT JOIN areas country_area ON country_area.id = CASE
              WHEN target_area.area_type_id = 4 THEN target_area.parent_id
              WHEN target_area.area_type_id = 3 THEN target_area.id
              ELSE NULL
            END
            LEFT JOIN areas subcont_area ON subcont_area.id = CASE
              WHEN target_area.area_type_id = 4 THEN country_area.parent_id
              WHEN target_area.area_type_id = 3 THEN target_area.parent_id
              WHEN target_area.area_type_id = 2 THEN target_area.id
              ELSE NULL
            END
            LEFT JOIN areas continent_area ON continent_area.id = CASE
              WHEN target_area.area_type_id = 4 THEN subcont_area.parent_id
              WHEN target_area.area_type_id = 3 THEN subcont_area.parent_id
              WHEN target_area.area_type_id = 2 THEN target_area.parent_id
              WHEN target_area.area_type_id = 1 THEN target_area.id
              ELSE NULL
            END
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
      categories: r.categories || [],
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
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'id', pc.id,
              'name', pc.name,
              'slug', pc.slug,
              'dimensionCode', cd.code,
              'dimensionName', cd.name
            ) ORDER BY cd.sort_order ASC, pc.sort_order ASC)
            FROM product_category_assignments pca
            INNER JOIN product_categories pc ON pc.id = pca.category_id
            INNER JOIN category_dimensions cd ON cd.id = pc.dimension_id
            WHERE pca.product_id = p.id AND pc.is_active = TRUE
          ),
          '[]'::json
        ) AS categories
      FROM product_variants v
      INNER JOIN products p ON p.id = v.product_id
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

    // Fetch upcoming departures with available capacity, age-band pricings, and itemized components
    const trips = await this.dataSource.query(`
      SELECT
        t.id, t.trip_code, t.start_date, t.end_date, t.status,
        t.min_quota, t.max_quota,
        (t.max_quota - COALESCE(b.booked_count, 0)) AS available_seats,
        EXISTS(SELECT 1 FROM product_itineraries pi WHERE pi.trip_id = t.id AND pi.is_active = TRUE) AS has_trip_override,
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'id', ptp.id,
              'ageBand', ptp.age_band,
              'minAge', ptp.min_age,
              'maxAge', ptp.max_age,
              'consumesQuota', ptp.consumes_quota,
              'basePrice', ptp.base_price,
              'sellingPrice', ptp.selling_price,
              'currency', ptp.currency,
              'components', COALESCE(
                (
                  SELECT json_agg(json_build_object(
                    'name', ppc.name,
                    'amount', ppc.amount,
                    'isIncluded', ppc.is_included,
                    'sortOrder', ppc.sort_order
                  ) ORDER BY ppc.sort_order ASC)
                  FROM product_pricing_components ppc
                  WHERE ppc.pricing_id = ptp.id
                ),
                '[]'::json
              )
            ) ORDER BY ptp.age_band ASC)
            FROM product_trip_pricings ptp
            WHERE ptp.trip_id = t.id
          ),
          '[]'::json
        ) AS pricings
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
        categories: variant.categories || [],
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

## 📊 Read-Optimized Seat Availability & Quota Indicators (Catalog Scope)

Real-time concurrency locks (`SELECT ... FOR UPDATE`) and transactional seat reservations are decoupled from the website catalog contract, belonging downstream to the Checkout/Booking domain.

For storefront discovery, PDP departure calendars, and "Where To?" search feeds, the backend computes nominal available seat indicators (`availableSeats = max_quota - booked_seats`):

```typescript
// services/trip-availability.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { DataSource } from 'typeorm';

export interface TripAvailabilitySummary {
  tripId: string;
  tripCode: string;
  startDate: string;
  endDate: string;
  status: 'ACTIVE' | 'FULL' | 'CANCELLED';
  minQuota: number;
  maxQuota: number;
  bookedSeats: number;
  availableSeats: number;
  isBookable: boolean;
  pricings: {
    ageBand: 'ADULT' | 'INFANT';
    sellingPrice: number;
    currency: string;
    consumesQuota: boolean;
  }[];
}

@Injectable()
export class TripAvailabilityService {
  constructor(private readonly dataSource: DataSource) {}

  /**
   * Retrieves active upcoming departures with nominal seat availability
   * and age-band pricing tiers for catalog display.
   */
  async getVariantTrips(variantId: string): Promise<TripAvailabilitySummary[]> {
    const query = `
      SELECT
        t.id,
        t.trip_code,
        t.start_date,
        t.end_date,
        t.status,
        t.min_quota,
        t.max_quota,
        COALESCE(b.booked_count, 0) AS booked_seats,
        GREATEST(0, t.max_quota - COALESCE(b.booked_count, 0)) AS available_seats,
        COALESCE(
          (
            SELECT json_agg(json_build_object(
              'ageBand', ptp.age_band,
              'sellingPrice', ptp.selling_price,
              'currency', ptp.currency,
              'consumesQuota', ptp.consumes_quota
            ) ORDER BY (CASE WHEN ptp.age_band = 'ADULT' THEN 1 ELSE 2 END))
            FROM product_trip_pricings ptp
            WHERE ptp.trip_id = t.id
          ),
          '[]'::json
        ) AS pricings
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
    `;

    const rows = await this.dataSource.query(query, [variantId]);

    return rows.map((r: any) => {
      const availableSeats = parseInt(r.available_seats, 10);
      return {
        tripId: r.id,
        tripCode: r.trip_code,
        startDate: r.start_date,
        endDate: r.end_date,
        status: r.status,
        minQuota: r.min_quota,
        maxQuota: r.max_quota,
        bookedSeats: parseInt(r.booked_seats, 10),
        availableSeats,
        isBookable: r.status === 'ACTIVE' && availableSeats > 0,
        pricings: r.pricings || [],
      };
    });
  }
}
```
