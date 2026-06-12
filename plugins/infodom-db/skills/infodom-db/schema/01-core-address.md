# Infodom DB — Core Address Tables

## `address_info_address` — ~7.97 M rows
Central entity. The `full_address` column is the natural unique key.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid | NO | PK |
| `country` | varchar(100) | NO | |
| `city` | varchar(100) | NO | |
| `street` | varchar(100) | NO | |
| `house_number` | varchar(10) | NO | |
| `postal_code` | varchar(10) | NO | |
| `full_address` | varchar(255) | NO | UNIQUE |
| `has_internet_access` | boolean | NO | |
| `address_point` | geometry (PostGIS) | NO | point location |
| `building_polygon` | geometry (PostGIS) | YES | building footprint |
| `district_id` | uuid FK → `address_info_district` | YES | |
| `estate_id` | uuid FK → `address_info_estate` | YES | |
| `pm_10_sensor_id` | uuid FK → `address_info_airquality` | YES | nearest PM10 sensor |
| `pm_2_5_sensor_id` | uuid FK → `address_info_airquality` | YES | nearest PM2.5 sensor |
| `stink_area_id` | uuid FK → `address_info_stinkarea` | YES | |

## `address_info_addressmatching` — ~7.96 M rows
One-to-one hub. Each address gets exactly one matching record; all 11 proximity match tables link through here.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `address_id` | uuid FK UNIQUE → `address_info_address` | |

## `address_info_district` — 35 rows
Administrative district polygons (e.g. Wrocław districts).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `name` | varchar(255) UNIQUE | NO |
| `polygon` | geometry | NO |
| `description` | text | NO |
| `area` | double precision | YES |
| `population` | integer | YES |

## `address_info_estate` — 825 rows
Sub-district neighbourhood polygons.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `name` | varchar(255) UNIQUE | NO |
| `polygons` | geometry | NO |

## `address_info_plot` — 37.7 M rows
Land registry (EGIB cadastre) plots. PK is the cadastre string identifier (not a UUID).  
Linked to `address_info_buildpermit` via `buildpermit.plot_id_id → plot.id`.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | varchar(100) | NO | PK — cadastre string |
| `plot_number` | varchar(100) | NO | |
| `district_name` | varchar(200) | YES | |
| `district_number` | varchar(100) | NO | |
| `unit_number` | varchar(100) | YES | |
| `municipality_name` | varchar(200) | YES | |
| `registration_field` | varchar(200) | YES | |
| `classutilities_EGIB` | varchar(200) | YES | land-use class |
| `registration_group` | varchar(200) | YES | |
| `date` | timestamptz | YES | |
| `polygon` | geometry | YES | plot boundary |

## `address_info_flat` — 0 rows (reserved for future use)

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `flat_number` | varchar(10) | NO |
| `size` | double precision | YES |
| `floor_number` | integer | YES |
| `description` | text | NO |
| `additional_rooms` | text | NO |
| `separateness` | boolean | YES |
| `address_id` | uuid FK UNIQUE → `address_info_address` | NO |

## `address_info_traveltime` — 0 rows (reserved for future use)

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `destination` | varchar(1000) | NO |
| `travel_type` | varchar(1000) | NO |
| `travel_time_per_hour` | jsonb | NO |
| `address_id` | uuid FK UNIQUE → `address_info_address` | YES |

## Query patterns

```sql
-- Look up an address by full text
SELECT * FROM address_info_address WHERE full_address = 'ul. Przykładowa 1, 50-000 Wrocław';

-- All addresses in a district
SELECT a.* FROM address_info_address a
JOIN address_info_district d ON a.district_id = d.id
WHERE d.name = 'Śródmieście';

-- Get the addressmatching hub id for a given address (needed to query match tables)
SELECT am.id AS matching_id
FROM address_info_addressmatching am
WHERE am.address_id = '<address_uuid>';
```
