---
name: infodom-db
description: This skill should be used when the user asks a question about the Infodom PostgreSQL database, needs a SQL query, wants to explore the database schema, or discusses data in tables — including addresses, address proximity / dynamic matching, users, user bindings, payments, orders, purchased reports, report credits, geo layers (amenities, flood zones, noise, air quality, transport stops), forms/questionnaires, auth tokens, or admin audit logs. Activate whenever the user mentions querying data, understanding table structure, writing SELECT statements, or any mention of Infodom data models.
version: 1.0.0
---

# Infodom Database Assistant

Database assistant for the **Infodom** production PostgreSQL database.

Read ONLY the relevant schema file(s) based on the user's question, then answer or write the SQL query. Do not read files that are not relevant.

## Schema files (read only what you need)

Schema files are bundled with this skill. When working in the `infodom-IaC` repository they are at `.claude/skills/infodom-db/schema/`. When installed as a plugin, look for them at the path where the plugin was installed.

| File | Read when the question involves… |
|---|---|
| `schema/00-overview.md` | General orientation, cross-domain relationships, connection details, naming conventions |
| `schema/01-core-address.md` | `address_info_address`, `district`, `estate`, `plot`, `flat`, `traveltime` |
| `schema/02-geo-layers.md` | Amenities, building permits, car accidents, hazard reports, monuments, nature points/polygons, flood zones, noise zones, air quality, transport stops, shelters, stink areas, parking zones |
| `schema/03-proximity-queries.md` | Proximity / "what's near this address" queries, distance filtering, radii; the dropped `address_info_address*match` tables |
| `schema/04-users-forms.md` | Users, address bindings, questionnaire categories/questions/submissions/answers |
| `schema/05-payments.md` | Orders, purchases, purchased reports, free-report credits and ledger |
| `schema/06-auth-admin.md` | Email auth, social login, MFA, admin audit logs, Celery Beat scheduler, Django internals |

## Rules

- **Read-only queries only.** Never suggest INSERT / UPDATE / DELETE / DROP / TRUNCATE / ALTER unless the user explicitly says they want to modify data — and warn them it is a production database.
- There are no match tables (`address_info_addressmatching`, `address_info_address*match` — dropped in backend 1.8.0, migration 0046). Compute proximity on the fly from one address: `ST_DWithin` degree prefilter on the raw geometry (uses GIST), then an exact `ST_DistanceSphere(...) <= metres` cutoff — see `03-proximity-queries.md`.
- Use `LIMIT` on exploratory queries.
- Monetary amounts are stored in pennies — divide by 100 for PLN.
- Geospatial columns are PostGIS `geometry` (SRID 4326) — use `ST_DWithin`, `ST_Contains`, `ST_Distance`, etc. Cast to `::geography` when you need metre-accurate distances.
- Status values for `form_formsubmission.status` and `users_useraddressbinding.status` are: `PENDING`, `VERIFIED`, `REJECTED`.

## Output format

1. Briefly confirm which schema file(s) you read.
2. Answer the question or provide the SQL.
3. If writing SQL: add a short comment explaining what it does and flag any performance consideration (e.g. if the query touches a very large table).
