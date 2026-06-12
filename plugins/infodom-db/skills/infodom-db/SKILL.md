---
name: infodom-db
description: This skill should be used when the user asks a question about the Infodom PostgreSQL database, needs a SQL query, wants to explore the database schema, or discusses data in tables — including addresses, address matching, users, user bindings, payments, orders, purchased reports, report credits, geo layers (amenities, flood zones, noise, air quality, transport stops), forms/questionnaires, auth tokens, or admin audit logs. Activate whenever the user mentions querying data, understanding table structure, writing SELECT statements, or any mention of Infodom data models.
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
| `schema/01-core-address.md` | `address_info_address`, `addressmatching`, `district`, `estate`, `plot`, `flat`, `traveltime` |
| `schema/02-geo-layers.md` | Amenities, building permits, car accidents, hazard reports, monuments, nature points/polygons, flood zones, noise zones, air quality, transport stops, shelters, stink areas, parking zones |
| `schema/03-match-tables.md` | Any of the 11 `address_info_address*match` tables, proximity queries, distance filtering |
| `schema/04-users-forms.md` | Users, address bindings, questionnaire categories/questions/submissions/answers |
| `schema/05-payments.md` | Orders, purchases, purchased reports, free-report credits and ledger |
| `schema/06-auth-admin.md` | Email auth, social login, MFA, admin audit logs, Celery Beat scheduler, Django internals |

## Rules

- **Read-only queries only.** Never suggest INSERT / UPDATE / DELETE / DROP / TRUNCATE / ALTER unless the user explicitly says they want to modify data — and warn them it is a production database.
- Always use `address_matching_id` as the first filter on any match table — these tables have hundreds of millions of rows.
- Use `LIMIT` on exploratory queries.
- Monetary amounts are stored in pennies — divide by 100 for PLN.
- Geospatial columns are PostGIS `geometry` — use `ST_DWithin`, `ST_Contains`, `ST_Distance`, etc. Cast to `::geography` when you need metre-accurate distances.
- Status values for `form_formsubmission.status` and `users_useraddressbinding.status` are: `PENDING`, `VERIFIED`, `REJECTED`.

## Output format

1. Briefly confirm which schema file(s) you read.
2. Answer the question or provide the SQL.
3. If writing SQL: add a short comment explaining what it does and flag any performance consideration (e.g. if the query touches a very large table).
