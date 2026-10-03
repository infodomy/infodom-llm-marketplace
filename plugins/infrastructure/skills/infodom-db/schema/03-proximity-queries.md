# Infodom DB — Proximity Queries (dynamic matching)

> **The pre-computed match tables no longer exist.** Backend migration
> `address_info.0046_remove_addressmatching_amenities_and_more` (backend 1.8.0) dropped
> `address_info_addressmatching` (the 1:1 hub) and all 11 `address_info_address*match` tables
> (amenity, buildpermit, caraccident, floodzone, hazardreport, monument, naturepoint, naturepolygon,
> noisezone, publictransportstop, shelter). There is no `address_matching_id` and no stored `distance`.
> Environments still on an older backend may have them until the migration runs — check with
> `SELECT tablename FROM pg_tables WHERE tablename LIKE 'address_info_%match%';`.

The backend now computes "address ↔ layer" proximity on the fly
(`infodom/address_info/logic/dynamic_matching/*.py` in `infodom-backend`). Write SQL the same way.

## How the backend computes it

All geometry columns are `geometry(…, 4326)` (lon/lat degrees). Per layer, for one address point:

1. **Business filters first** (e.g. amenity `category`, stop `stop_type`, accident casualties > 0).
2. **Index prefilter**: `ST_DWithin(layer.geom, addr.address_point, <deg>)` on the raw `geometry`
   (no `::geography` cast, so the GIST index is used). `<deg>` is a *conservative* degree radius:
   `GREATEST(r_km / 111.0, r_km / (111.0 * GREATEST(ABS(COS(RADIANS(lat))), 0.01)))` — i.e. the
   larger of the N-S and E-W degree spans at the address latitude, so it over-fetches slightly.
3. **Exact metre cutoff**: distance in metres (GeoDjango `Distance` on a geodetic `geometry` field →
   `ST_DistanceSphere`) `<= r_m`, computed only on the prefiltered rows; then `ORDER BY distance`.

Radii (`config/settings/app.py`): `MINIMAL_RADIUS` 50 m · `SMALL_RADIUS` 200 m · `MEDIUM_RADIUS` 1000 m ·
`BIG_RADIUS` 5000 m.

| Layer table | Geometry column | Radius | Extra rules |
|---|---|---|---|
| `address_info_amenity` | `location` | MEDIUM; BIG for `subcategory = 'hospital'` | filtered by `category` |
| `address_info_buildpermit` | `address_info_plot.polygon` via `plot_id_id` | MEDIUM | ordered by decision year desc, then distance |
| `address_info_caraccident` | `location` | MEDIUM | only rows with `light_/heavy_/fatal_casualties > 0` |
| `address_info_floodzone` | `polygon` | SMALL | |
| `address_info_hazardreport` | `location` | MEDIUM | optional `status` / `hazard_type` filters |
| `address_info_monument` | `location` | MEDIUM | |
| `address_info_naturepoint` | `location` | MEDIUM | |
| `address_info_naturepolygon` | `polygons` | MEDIUM; BIG for `type IN ('national_park','landscape_park')` | |
| `address_info_noisezone` | `polygon` | MINIMAL | |
| `address_info_publictransportstop` | `address_point` (nullable) | MEDIUM; BIG for `stop_type IN ('train','metro')` | skip `address_point IS NULL` |
| `address_info_shelter` | `location` | MEDIUM | |
| `address_info_parkingzone` | `polygon` | MEDIUM | |

## Query rules

- Anchor on **one address** (or a small set) — never cross-join a layer against all 8.5 M addresses.
- Keep `ST_DWithin` on the raw `geometry` with the degree radius so the GIST index is used; casting to
  `::geography` inside `ST_DWithin` skips the geometry index.
- Apply the exact metre filter (`ST_DistanceSphere(...) <= r_m`) after the prefilter.
- Build permits: join `address_info_plot` (16 GB, 37.7 M rows) on `p.id = bp.plot_id_id` — filter by
  plot polygon proximity, not by scanning permits.

## Template

```sql
-- <layer> rows within :r_m metres of one address, nearest first (mirrors dynamic_matching)
WITH addr AS (
  SELECT address_point AS pt,
         GREATEST(:r_m / 1000.0 / 111.0,
                  :r_m / 1000.0 / (111.0 * GREATEST(ABS(COS(RADIANS(ST_Y(address_point)))), 0.01))) AS deg
  FROM address_info_address
  WHERE full_address = '<full address>'          -- or id = '<address_uuid>'
)
SELECT e.*, ST_DistanceSphere(e.location, addr.pt) AS distance_m
FROM address_info_<layer> e, addr
WHERE ST_DWithin(e.location, addr.pt, addr.deg)  -- GIST prefilter (degrees)
  AND ST_DistanceSphere(e.location, addr.pt) <= :r_m
ORDER BY distance_m
LIMIT 50;
```

## Worked examples

```sql
-- Nearest 5 amenities (non-hospital) within 1 km of an address
WITH addr AS (
  SELECT address_point AS pt,
         GREATEST(1.0/111.0, 1.0/(111.0*GREATEST(ABS(COS(RADIANS(ST_Y(address_point)))),0.01))) AS deg
  FROM address_info_address WHERE id = '<uuid>'
)
SELECT am.category, am.subcategory, ST_DistanceSphere(am.location, addr.pt) AS distance_m
FROM address_info_amenity am, addr
WHERE am.subcategory <> 'hospital'
  AND ST_DWithin(am.location, addr.pt, addr.deg)
  AND ST_DistanceSphere(am.location, addr.pt) <= 1000
ORDER BY distance_m LIMIT 5;

-- Car accidents with casualties within 200 m of an address
WITH addr AS (
  SELECT address_point AS pt,
         GREATEST(0.2/111.0, 0.2/(111.0*GREATEST(ABS(COS(RADIANS(ST_Y(address_point)))),0.01))) AS deg
  FROM address_info_address WHERE id = '<uuid>'
)
SELECT ca.*, ST_DistanceSphere(ca.location, addr.pt) AS distance_m
FROM address_info_caraccident ca, addr
WHERE (ca.light_casualties > 0 OR ca.heavy_casualties > 0 OR ca.fatal_casualties > 0)
  AND ST_DWithin(ca.location, addr.pt, addr.deg)
  AND ST_DistanceSphere(ca.location, addr.pt) <= 200
ORDER BY distance_m;

-- Bus & tram stops within 300 m (stop_type lives on the stop row itself)
WITH addr AS (
  SELECT address_point AS pt,
         GREATEST(0.3/111.0, 0.3/(111.0*GREATEST(ABS(COS(RADIANS(ST_Y(address_point)))),0.01))) AS deg
  FROM address_info_address WHERE id = '<uuid>'
)
SELECT s.name, s.stop_type, ST_DistanceSphere(s.address_point, addr.pt) AS distance_m
FROM address_info_publictransportstop s, addr
WHERE s.address_point IS NOT NULL
  AND s.stop_type IN ('bus', 'tram')
  AND ST_DWithin(s.address_point, addr.pt, addr.deg)
  AND ST_DistanceSphere(s.address_point, addr.pt) <= 300
ORDER BY distance_m;

-- Build permits on plots within 1 km, newest decision year first
WITH addr AS (
  SELECT address_point AS pt,
         GREATEST(1.0/111.0, 1.0/(111.0*GREATEST(ABS(COS(RADIANS(ST_Y(address_point)))),0.01))) AS deg
  FROM address_info_address WHERE id = '<uuid>'
)
SELECT bp.*, ST_DistanceSphere(p.polygon, addr.pt) AS distance_m
FROM address_info_buildpermit bp
JOIN address_info_plot p ON p.id = bp.plot_id_id, addr
WHERE ST_DWithin(p.polygon, addr.pt, addr.deg)
  AND ST_DistanceSphere(p.polygon, addr.pt) <= 1000
ORDER BY EXTRACT(YEAR FROM bp.decision_date) DESC, distance_m;
```
