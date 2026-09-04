# Area Domain - Technical Data Model & Architecture

> **Overview**
> Technical documentation for the Area/Geography Domain data model. This architecture is centered around managing hierarchical geographical regions, destinations, and administrative boundaries (Continents, Countries, and Cities — with maximum depth strictly capped at City level) to support multi-level location mapping across trips, products, and hotels.
>
> _Engineered for High Scalability, Hierarchical Tree Traversal, and optimized for a NestJS + PostgreSQL stack._

> **See Also:**
> - [Product Technical Design](./product-technical-design.md) — Cross-domain `product_locations` reference
> - [Search & Filter Architecture](./product-search-filter-technical-design.md) — Area hierarchy joins in search SQL
> - [SEO Technical Design](./seo-technical-design.md) — Destination landing page SEO (`target_type = 'AREA'`)
> - [Area Domain Contracts](../contracts/area-contract.md) — API endpoints, autocomplete, destination landing
> - [Backend Guide](../backend/area-backend-guide.md) — Recursive CTE traversal, PostGIS spatial queries, and caching
> - [Frontend Guide](../frontend/area-frontend-guide.md) — "Where To?" autocomplete widget and destination landing pages

---

## 🏗️ Architecture, Scalability & Engineering Principles

The following architectural guidelines must be strictly adhered to during implementation to ensure enterprise-grade reliability, optimal client consumption, and seamless deployment.

### 1. Hierarchical Closure Pattern & Adjacency List (Continent → Country → City)

The geography tree is strictly capped at a 3-tier hierarchy: **Continent → Country → City** (maximum granularity ends at City level; no districts or sub-districts). To handle multi-level geographical relationships efficiently without recursive CTE performance hits on deep reads, the architecture combines an **Adjacency List (`parent_id`)** for simple CRUD operations with composite indexing for rapid subtree lookups.

### 2. DevOps & Automated Migrations

The provided PostgreSQL DDL scripts serve as the foundational schema. These migration scripts must be version-controlled and integrated directly into CI/CD deployment pipelines to maintain consistent geographic reference data states across staging and production environments.

### 3. Handling Polymorphic & Spatial Extensions

Areas act as anchor points for multiple entities (`products`, `product_locations`, `hotels`, `airports`).

- **Geospatial Indexing:** We utilize PostgreSQL's **PostGIS** extension (`GEOMETRY` / `GEOGRAPHY` types) alongside spatial B-Tree/GiST indexes (`USING GIST`) to support high-performance radius searches and boundary containment queries (e.g., "Find all products within a 50km radius of Central Jakarta").
- **Denormalized Redundancy:** To minimize complex multi-table joins during high-traffic product catalog rendering, child records (such as `product_locations`) store denormalized metadata (`area_name`, `country_code`) while maintaining strict foreign-key references to `areas.id`.

### 4. Concurrency & Reference Data Integrity

Geographic master data is largely read-heavy and updated infrequently by administrative jobs. Implement **Read-Through Caching** (via Redis) in NestJS for area hierarchies and popular destination nodes to reduce database load to near-zero for search widget autocomplete endpoints.

### 5. Precision & Geospatial Standards

Geographic coordinates (`lat`, `lng`) are strictly cast as `DOUBLE PRECISION` (or `GEOMETRY(Point, 4326)` using standard WGS 84 spatial reference systems). Bounding boxes and polygons use standard GeoJSON formatting for seamless frontend map rendering (Mapbox / Google Maps integration).

---

## 🛠️ PostgreSQL DDL Migration Script

Use this schema as the baseline for your ORM migrations.

```sql
-- =========================================================================
-- 1. EXTENSIONS & SPATIAL SETUP
-- =========================================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis"; -- Required for advanced spatial queries

-- =========================================================================
-- 2. AREA / GEOGRAPHY CORE TABLES
-- =========================================================================
CREATE TABLE area_types (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE, -- 'CONTINENT' | 'COUNTRY' | 'CITY' (max granularity is CITY)
    description TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT chk_area_types_name CHECK (name IN ('CONTINENT', 'COUNTRY', 'CITY'))
);

CREATE TABLE areas (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    parent_id UUID REFERENCES areas(id) ON DELETE CASCADE, -- Adjacency list for hierarchy
    area_type_id INT NOT NULL REFERENCES area_types(id) ON DELETE RESTRICT,
    code VARCHAR(50) UNIQUE NOT NULL, -- e.g., 'ID-JK', 'JAPAN-TYO'
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    iso_code VARCHAR(10), -- ISO 3166-1 alpha-2 for countries, etc.
    lat DOUBLE PRECISION,
    lng DOUBLE PRECISION,
    boundary GEOMETRY(Polygon, 4326), -- PostGIS polygon for geographic boundaries
    listing_status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE | INACTIVE | ARCHIVED
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP NULL,

    CONSTRAINT chk_areas_listing_status CHECK (listing_status IN ('ACTIVE', 'INACTIVE', 'ARCHIVED'))
);

-- COMPOSITE & SPATIAL INDEXES
CREATE INDEX idx_areas_parent_id ON areas(parent_id);
CREATE INDEX idx_areas_type ON areas(area_type_id);
CREATE INDEX idx_areas_boundary_gist ON areas USING GIST (boundary);
CREATE INDEX idx_areas_slug ON areas(slug);

-- =========================================================================
-- 3. AUDIT TRIGGER AUTOMATION
-- =========================================================================
CREATE OR REPLACE FUNCTION set_updated_at_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_area_types_updated_at BEFORE UPDATE ON area_types FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
CREATE TRIGGER trg_areas_updated_at      BEFORE UPDATE ON areas      FOR EACH ROW EXECUTE FUNCTION set_updated_at_timestamp();
```

---

## 📊 Domain Data Scenario

**Area Hierarchy:** Asia (Continent) -> Japan (Country) -> Tokyo (City)
**Root Area ID:** `area_asia_01`

_(Sample data is included below each respective ERD block to illustrate context)._

### 1. Core Area & Cross-Domain Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    area_types ||--o{ areas : "area_type_id"
    areas ||--o{ areas : "parent_id (Continent -> Country -> City)"
    areas ||--o{ product_locations : "area_id (City destination marker)"
    products ||--o{ product_locations : "product_id"

    area_types {
        int       id           PK
        varchar   name         "CONTINENT | COUNTRY | CITY"
        text      description
        timestamp created_at
        timestamp updated_at
    }

    areas {
        uuid      id             PK
        uuid      parent_id      FK "self-reference (Continent -> Country -> City)"
        int       area_type_id   FK "references area_types.id"
        varchar   code           "e.g. ASIA, JP, JP-TYO"
        varchar   name           "e.g. Asia, Japan, Tokyo"
        varchar   slug           "e.g. asia, japan, tokyo"
        varchar   iso_code       "ISO 3166-1 alpha-2 (JP)"
        double    lat
        double    lng
        geometry  boundary       "PostGIS Polygon (4326)"
        varchar   listing_status "ACTIVE | INACTIVE | ARCHIVED"
        int       sort_order
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }

    product_locations {
        uuid      id          PK
        uuid      product_id  FK
        varchar   source_type "AREA | MANUAL"
        uuid      area_id     FK "logical FK → areas.id (City level)"
        varchar   area_name   "denormalized city name"
        double    lat
        double    lng
        text      address
        int       sort_order
        timestamp created_at
        timestamp updated_at
    }

    products {
        uuid      id             PK
        varchar   code           "e.g. TURKEY-WONDERS, GWE"
        varchar   slug           "e.g. grand-west-europe"
        varchar   product_type   "JOURNEY | OPEN_TRIP | PRIVATE_TRIP | DAY_TOUR"
        varchar   listing_status "DRAFT | PENDING_REVIEW | ACTIVE | INACTIVE | ARCHIVED | SUSPENDED"
    }
```

**Sample Data**

> _(Note: Standard audit timestamps `created_at`, `updated_at`, and `deleted_at` are defined in the schema and ERD above, but omitted from the sample data tables below for readability)._

**`area_types`**

| id | name | description |
| :--- | :--- | :--- |
| 1 | CONTINENT | Global continental landmasses and geographic macro-regions (root level) |
| 2 | COUNTRY | Sovereign states and independent nations |
| 3 | CITY | Major metropolitan areas, municipalities, and primary tour destinations (maximum granularity) |

**`areas`**

| id | parent_id | area_type_id | code | name | slug | iso_code | lat | lng | boundary | listing_status | sort_order |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| area_asia_01 | NULL | 1 | ASIA | Asia | asia | NULL | 34.0479 | 100.6197 | NULL | ACTIVE | 1 |
| area_jp_01 | area_asia_01 | 2 | JP | Japan | japan | JP | 36.2048 | 138.2529 | NULL | ACTIVE | 2 |
| area_tyo_01 | area_jp_01 | 3 | JP-TYO | Tokyo | tokyo | NULL | 35.6762 | 139.6503 | NULL | ACTIVE | 3 |

**`product_locations` (Cross-Domain Mapping Sample)**

| id | product_id | source_type | area_id | area_name | lat | lng | address | sort_order |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| loc_jp_01 | prod_japan_01 | AREA | area_tyo_01 | Tokyo | 35.6762 | 139.6503 | Shinjuku & Shibuya, Tokyo, Japan | 1 |

---

### 2. 3-Tier Area Hierarchy ERD (Conceptual Level View)

This conceptual ERD illustrates how the 3 geographical tiers relate hierarchically via `parent_id` and terminate at the `City` level where tour products anchor their destination stops:

```mermaid
erDiagram
    CONTINENT ||--o{ COUNTRY : "parent_id (1:N)"
    COUNTRY   ||--o{ CITY    : "parent_id (1:N)"
    CITY      ||--o{ product_locations : "area_id (1:N City destination)"
    products  ||--o{ product_locations : "product_id (1:N)"

    CONTINENT {
        uuid    id          PK "e.g. area_asia_01"
        varchar name        "Asia"
        varchar code        "ASIA"
        int     area_type   "1 (CONTINENT - root)"
    }

    COUNTRY {
        uuid    id          PK "e.g. area_jp_01"
        uuid    parent_id   FK "references CONTINENT.id"
        varchar name        "Japan"
        varchar iso_code    "JP"
        int     area_type   "2 (COUNTRY)"
    }

    CITY {
        uuid    id          PK "e.g. area_tyo_01"
        uuid    parent_id   FK "references COUNTRY.id"
        varchar name        "Tokyo"
        varchar code        "JP-TYO"
        int     area_type   "3 (CITY - Maximum Depth)"
    }

    product_locations {
        uuid    id          PK "Destination stop"
        uuid    product_id  FK "references products.id"
        uuid    area_id     FK "logical FK → CITY.id"
        varchar area_name   "Tokyo (denormalized)"
    }

    products {
        uuid    id          PK "Tour product"
        varchar code        "e.g. JAPAN-AUTUMN"
        varchar slug        "japan-autumn"
    }
```

---

### 3. High-Level Global Area Hierarchy Tree (Architecture Flowchart)

```mermaid
flowchart TB

    ATYPE["area_types<br/>1: CONTINENT<br/>2: COUNTRY<br/>3: CITY"]

    subgraph AH["Area Hierarchy Tree (Capped at City Level)"]
        ROOT["areas<br/>Continent: Asia<br/>id: area_asia_01<br/>parent_id: NULL"]

        COUNTRY["areas<br/>Country: Japan<br/>id: area_jp_01<br/>parent_id: area_asia_01"]

        CITY["areas<br/>City: Tokyo<br/>id: area_tyo_01<br/>parent_id: area_jp_01"]
    end

    ATYPE -. "area_type_id = 1" .-> ROOT
    ATYPE -. "area_type_id = 2" .-> COUNTRY
    ATYPE -. "area_type_id = 3" .-> CITY

    ROOT -->|"parent_id"| COUNTRY
    COUNTRY -->|"parent_id"| CITY

    PROD["products<br/>Tours / Journeys"]

    PROD_LOC["product_locations<br/>Destination Markers"]

    PROD -->|"product_id (1:N)"| PROD_LOC

    CITY -->|"area_id (Logical FK to areas.id)"| PROD_LOC
```
