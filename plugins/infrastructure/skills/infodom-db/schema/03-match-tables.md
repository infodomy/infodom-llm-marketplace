# Infodom DB — Match Tables

Match tables are pre-computed proximity joins between `address_info_address` (via `address_info_addressmatching`) and each geospatial layer. They are the **largest tables in the database**.

## Pattern (all 11 tables share this structure)

```
id                   uuid PK
distance             double precision   — metres from address point to entity
address_matching_id  uuid FK → address_info_addressmatching  (UNIQUE composite)
<entity>_id          uuid FK → <entity table>                 (UNIQUE composite)
```

The pair `(address_matching_id, <entity>_id)` is always unique.

## Table catalogue

| Table | ~Rows | Entity FK target | Extra columns |
|---|---|---|---|
| `address_info_addressamenitymatch` | 151 M | `address_info_amenity` | — |
| `address_info_addressbuildpermitmatch` | **482 M** | `address_info_buildpermit` | — |
| `address_info_addresscaraccidentmatch` | 68.6 M | `address_info_caraccident` | — |
| `address_info_addressfloodzonematch` | 2.3 M | `address_info_floodzone` | — |
| `address_info_addresshazardreportmatch` | 36.8 M | `address_info_hazardreport` | — |
| `address_info_addressmonumentmatch` | 26.9 M | `address_info_monument` | — |
| `address_info_addressnaturepointmatch` | 17.4 M | `address_info_naturepoint` | — |
| `address_info_addressnaturepolygonmatch` | 4.2 M | `address_info_naturepolygon` | — |
| `address_info_addressnoisezonematch` | 151 K | `address_info_noisezone` | — |
| `address_info_addresspublictransportstopmatch` | 56.9 M | `address_info_publictransportstop` | `stop_type varchar(16)` — denormalised on the match row |
| `address_info_addresssheltermatch` | 98.6 M | `address_info_shelter` | — |

## Critical query rules

1. **Always start from `address_matching_id`** — it is the indexed UNIQUE column. Never full-scan a match table.
2. Queries should follow this join chain:

```sql
-- Template: get all <entities> near a given address, ordered by distance
SELECT e.*, m.distance
FROM address_info_<entity>match m
JOIN address_info_<entity> e ON e.id = m.<entity>_id
WHERE m.address_matching_id = (
    SELECT id FROM address_info_addressmatching WHERE address_id = '<address_uuid>'
)
ORDER BY m.distance
LIMIT 20;
```

3. `address_info_addressbuildpermitmatch` (482 M rows) is the heaviest — always add a `distance < X` predicate.
4. `address_info_addresspublictransportstopmatch.stop_type` is denormalised for read performance; it mirrors `address_info_publictransportstop.stop_type`.

## Worked examples

```sql
-- Nearest 5 amenities to a known address
SELECT am.category, am.subcategory, match.distance
FROM address_info_addressamenitymatch match
JOIN address_info_amenity am ON am.id = match.amenity_id
WHERE match.address_matching_id = (
    SELECT id FROM address_info_addressmatching WHERE address_id = '<uuid>'
)
ORDER BY match.distance LIMIT 5;

-- Count accidents within 200m of every address in Śródmieście
SELECT addr.full_address, COUNT(*) AS accidents_nearby
FROM address_info_address addr
JOIN address_info_district d ON addr.district_id = d.id AND d.name = 'Śródmieście'
JOIN address_info_addressmatching am ON am.address_id = addr.id
JOIN address_info_addresscaraccidentmatch acm ON acm.address_matching_id = am.id
WHERE acm.distance < 200
GROUP BY addr.full_address
ORDER BY accidents_nearby DESC;

-- Bus & tram stops within 300m, using denormalised stop_type
SELECT pt.name, pt.stop_type, m.distance
FROM address_info_addresspublictransportstopmatch m
JOIN address_info_publictransportstop pt ON pt.id = m.public_transport_stop_id
WHERE m.address_matching_id = (
    SELECT id FROM address_info_addressmatching WHERE address_id = '<uuid>'
)
  AND m.distance < 300
ORDER BY m.distance;
```
