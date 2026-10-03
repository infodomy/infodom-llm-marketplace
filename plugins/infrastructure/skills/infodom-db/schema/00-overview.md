# Infodom DB — Overview & Relationships

> PostgreSQL + PostGIS on AWS RDS  
> Host: `prod-locumo.crb5pcoqgbhd.eu-central-1.rds.amazonaws.com`, database: `infodom`  
> Connection: `PGPASSWORD="***REDACTED***" psql -h prod-locumo.crb5pcoqgbhd.eu-central-1.rds.amazonaws.com -d infodom -U postgres`

## What the app does

Infodom is a Polish property-intelligence platform. A user registers the address they live at, uploads proof of residence, and submits a verification questionnaire. On approval they receive a free environment report covering: noise, flood risk, air quality, nature, amenities, public transport, car accidents, building permits, civil-defence shelters, monuments, and odour zones.  
Reports can also be purchased directly with payment.

## Django app → DB prefix map

| Prefix | Domain | Schema file |
|---|---|---|
| `address_info_` | Addresses + geospatial layers (proximity computed on the fly) | `01-core-address.md`, `02-geo-layers.md`, `03-proximity-queries.md` |
| `users_` | Custom user + address binding | `04-users-forms.md` |
| `form_` | Verification questionnaire | `04-users-forms.md` |
| `payment_` | Orders, purchases, free-report credits | `05-payments.md` |
| `admin_panel_` | Admin audit logs | `06-auth-admin.md` |
| `account_` / `socialaccount_` / `authtoken_` / `mfa_` | django-allauth, DRF token, MFA | `06-auth-admin.md` |
| `auth_` / `django_*` | Django internals | `06-auth-admin.md` |
| `django_celery_beat_*` | Periodic scheduler | `06-auth-admin.md` |

## Key relationships (simplified)

```
users_user (66 rows)
  ├─► users_useraddressbinding ──────────────────► address_info_address (8.57 M)
  ├─► form_formsubmission ─────────────────────► address_info_address
  │     └─► form_questionanswer ──► form_question ──► form_category
  ├─► payment_userfreereports
  ├─► payment_freereportstransaction
  ├─► payment_userpurchasedreport ─────────────► address_info_address
  └─► payment_order ──► payment_purchase
                └─────────────────────────────► address_info_address

address_info_address (8.57 M)
  ├┄┄ [geospatial layer tables] — no FK; matched by PostGIS proximity at query time
  ├──► address_info_district (35)
  ├──► address_info_estate (825)
  ├──► address_info_airquality ×2 (PM10 sensor, PM2.5 sensor)
  └──► address_info_stinkarea (3)
```

## General conventions

- All PKs are **UUID** except: `address_info_plot.id` (varchar cadastre key), Django-internal tables (integer/bigint).
- Geospatial columns show as `USER-DEFINED` in `information_schema` — they are PostGIS `geometry` types.
- No pre-computed match tables: `address_info_addressmatching` and the 11 `address_info_address*match` tables were dropped in backend 1.8.0 (migration `address_info.0046`). Address↔layer proximity is computed on the fly — see `03-proximity-queries.md`.
- Monetary amounts are stored as **integers in pennies** (divide by 100 for PLN).
- Status fields (`PENDING` / `VERIFIED` / `REJECTED`) drive the admin approval workflow for both form submissions and address bindings.

## Size (dev, 2026-10, after migration 0046)

Total ~24 GB (was ~283 GB with the match tables). Largest tables: `address_info_plot` 16 GB ·
`address_info_address` 5.7 GB · `address_info_airquality` ~1 GB · `address_info_buildpermit` ~890 MB ·
`address_info_caraccident` ~580 MB.
