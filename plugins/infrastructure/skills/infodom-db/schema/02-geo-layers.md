# Infodom DB — Geospatial Layer Tables

These are the "entity" tables that hold the actual geospatial data. They are linked to addresses through the match tables (`address_info_address*match`). See `03-match-tables.md` for the match table schemas.

## `address_info_amenity` — 397 K rows
Points of interest from OpenStreetMap.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `category` | varchar(32) | NO | e.g. `shop`, `leisure`, `healthcare` |
| `subcategory` | varchar(64) | NO | composite UNIQUE with location |
| `location` | geometry | NO | OSM point |
| `details` | jsonb | NO | raw OSM tags |
| `external_id` | bigint | YES | UNIQUE — OSM node/way id |
| `district_id` | uuid FK → `address_info_district` | YES | |
| `estate_id` | uuid FK → `address_info_estate` | YES | |

## `address_info_buildpermit` — 1.59 M rows
Building permits from the national GUNB register.  
Linked to land plots via `plot_id_id` (string FK to `address_info_plot.id`).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `plot_id_id` | varchar(100) UNIQUE FK → `address_info_plot` | NO |
| `authority_name` | text | YES |
| `authority_address` | text | YES |
| `application_date` | timestamptz | NO |
| `decision_date` | timestamptz | NO |
| `investor_name` | text | YES |
| `voivodeship` | text | YES |
| `city` | text | YES |
| `investition_type` | text | YES |
| `category` | text | YES |
| `construction_explanation_name` | text | YES |
| `construction_type` | text | YES |
| `district_number` | text | YES |
| `plot_number` | text | YES |

## `address_info_caraccident` — 2.97 M rows
Polish police road accident database.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `source_id` | bigint UNIQUE | NO |
| `date` | timestamptz | NO |
| `light_casualties` | integer | NO |
| `heavy_casualties` | integer | NO |
| `fatal_casualties` | integer | NO |
| `location` | geometry | NO |

## `address_info_hazardreport` — 89 K rows
Urban hazard reports (city open-data portals).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `source_object_id` | bigint UNIQUE | NO |
| `hazard_type` | varchar(64) | NO |
| `status` | varchar(32) | NO |
| `date` | timestamptz | NO |
| `location` | geometry | NO |

## `address_info_monument` — 61 K rows
Heritage monuments from the national register.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `name` | varchar(500) UNIQUE | NO |
| `chronology` | varchar(500) | NO |
| `function` | varchar(500) | NO |
| `building_material` | varchar(500) | NO |
| `architectural_style` | varchar(500) | NO |
| `link` | varchar(500) | NO |
| `location` | geometry | NO |
| `address_id` | uuid FK → `address_info_address` | YES |
| `district_id` | uuid FK | YES |
| `estate_id` | uuid FK | YES |

## `address_info_naturepoint` — 114 K rows
Individual trees, benches, and other nature objects.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `external_id` | varchar UNIQUE | NO |
| `type` | varchar(500) | NO |
| `name` | varchar(500) | NO |
| `species` | varchar(500) | NO |
| `location` | geometry | NO |
| `district_id` | uuid FK | YES |
| `estate_id` | uuid FK | YES |

## `address_info_naturepolygon` — 9 K rows
Parks, forests, green areas (polygon geometries).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `external_id` | varchar UNIQUE | NO |
| `type` | varchar(500) | NO |
| `name` | varchar(500) | NO |
| `polygons` | geometry | NO |

M2M to districts: `address_info_naturepolygon_districts` (`naturepolygon_id`, `district_id`)  
M2M to estates: `address_info_naturepolygon_estates` (`naturepolygon_id`, `estate_id`)

## `address_info_floodzone` — 3.5 K rows
Flood risk zones from ISOK (national flood hazard maps).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `name` | varchar(255) | NO |
| `flood_type` | varchar(10) | NO |
| `probability` | double precision | NO |
| `polygon` | geometry | NO |
| `external_id` | varchar(255) UNIQUE | NO |

## `address_info_noisezone` — 70 rows
Acoustic contour polygons (aircraft, road, rail noise maps). Low row count — each row is one noise band polygon.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `noise_type` | varchar(20) | NO | composite UNIQUE |
| `time_period` | varchar(10) | NO | `D` = day, `N` = night |
| `decibel_lower` | double precision | NO | composite UNIQUE |
| `decibel_upper` | double precision | NO | composite UNIQUE |
| `polygon_hash` | varchar(128) UNIQUE | NO | dedup hash |
| `polygon` | geometry | NO | |

## `address_info_airquality` — 3.26 M rows
Interpolated air quality grid — one row per (location, pollution_type). Monthly averages stored as separate columns.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `coord_point` | geometry | NO | composite UNIQUE with pollution_type |
| `pollution_type` | varchar(10) | NO | `PM10`, `PM2_5`, etc. |
| `january` … `december` | double precision | YES | 12 monthly average columns |
| `interpolated` | boolean | NO | |

## `address_info_publictransportstop` — 130 K rows

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `name` | varchar(255) | NO |
| `stop_type` | varchar(16) | NO |
| `address_point` | geometry | YES |
| `district_id` | uuid FK | YES |
| `estate_id` | uuid FK | YES |

## `address_info_shelter` — 152 K rows
Civil defence shelters (MSWiA register).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `external_id` | integer UNIQUE | NO |
| `object_id` | integer | NO |
| `investigation_type` | varchar(50) | NO |
| `access_type` | varchar(50) | NO |
| `area` | double precision | NO |
| `capacity` | integer | NO |
| `subjective_rating` | integer | NO |
| `object_type` | varchar(50) | NO |
| `purpose` | varchar(50) | NO |
| `voivodeship` | varchar(100) | NO |
| `county` | varchar(100) | NO |
| `address` | varchar(255) | YES |
| `location` | geometry | NO |

## `address_info_stinkarea` — 3 rows
Odour nuisance zones.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `city` | varchar(100) | NO |
| `stink_level` | varchar(10) | NO |
| `polygons` | geometry UNIQUE | NO |

M2M to districts: `address_info_stinkarea_districts`  
M2M to estates: `address_info_stinkarea_estates`

## `address_info_parkingzone` — 0 rows (not yet populated)

| Column | Type |
|---|---|
| `id` | uuid PK |
| `name` | varchar(500) UNIQUE |
| `polygon` | geometry |
| `description` | text |

## Geospatial query patterns

```sql
-- Find all amenities within 500m of an address point
SELECT a.category, a.subcategory, a.details,
       ST_Distance(a.location::geography, addr.address_point::geography) AS distance_m
FROM address_info_amenity a, address_info_address addr
WHERE addr.full_address = 'ul. Przykładowa 1, 50-000 Wrocław'
  AND ST_DWithin(a.location::geography, addr.address_point::geography, 500)
ORDER BY distance_m;

-- Check if an address falls inside a flood zone
SELECT fz.name, fz.flood_type, fz.probability
FROM address_info_floodzone fz
JOIN address_info_address addr ON ST_Within(addr.address_point, fz.polygon)
WHERE addr.id = '<address_uuid>';
```
