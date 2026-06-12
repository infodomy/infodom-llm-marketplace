# Infodom DB — Auth, Admin & Django Internals

## Admin Panel Audit Logs

### `admin_panel_manageformsactionlog` — 30 rows
Records every admin action taken on a form submission (approve, reject, add note, etc.).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `admin_user_id` | uuid FK → `users_user` | YES |
| `form_submission_id` | uuid FK → `form_formsubmission` | YES |
| `action` | varchar(1000) | NO |
| `note` | text | NO |
| `created_at` | timestamptz | NO |

### `admin_panel_manageuseraddressbindingactionlog` — 18 rows
Records every admin action taken on a user address binding (approve, reject, etc.).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `admin_user_id` | uuid FK → `users_user` | YES |
| `user_id` | uuid FK → `users_user` | NO |
| `user_address_binding_id` | uuid FK → `users_useraddressbinding` | YES |
| `action` | varchar(1000) | NO |
| `note` | text | NO |
| `created_at` | timestamptz | NO |

---

## Authentication — django-allauth

### `account_emailaddress` — 65 rows
Tracks verified/primary email addresses per user.

| Column | Type |
|---|---|
| `id` | integer PK |
| `user_id` | uuid FK → `users_user` |
| `email` | varchar(254) UNIQUE |
| `verified` | boolean |
| `primary` | boolean |

### `account_emailconfirmation` — 32 rows
Pending email confirmation tokens.

| Column | Type |
|---|---|
| `id` | integer PK |
| `email_address_id` | integer FK → `account_emailaddress` |
| `key` | varchar(64) UNIQUE |
| `created` / `sent` | timestamptz |

### `socialaccount_socialaccount` — 35 rows
Social login connections (Google OAuth etc.).

| Column | Type | Notes |
|---|---|---|
| `id` | integer PK | |
| `provider` | varchar(200) | composite UNIQUE with uid |
| `uid` | varchar(191) | provider user id |
| `last_login` / `date_joined` | timestamptz | |
| `extra_data` | jsonb | raw provider payload |
| `user_id` | uuid FK → `users_user` | |

Other social tables (all empty):
- `socialaccount_socialapp` — OAuth app credentials
- `socialaccount_socialapp_sites` — M2M
- `socialaccount_socialtoken` — stored OAuth tokens

### `authtoken_token` — 0 rows
DRF token auth (not currently in use).

| Column | Type |
|---|---|
| `key` | varchar(40) PK |
| `user_id` | uuid FK UNIQUE |
| `created` | timestamptz |

### `mfa_authenticator` — 0 rows
MFA devices (TOTP etc.) — configured but not yet in use.

| Column | Type |
|---|---|
| `id` | bigint PK |
| `user_id` | uuid FK |
| `type` | varchar(20) |
| `data` | jsonb |
| `created_at` / `last_used_at` | timestamptz |

---

## Django Internals

| Table | Purpose |
|---|---|
| `auth_group` | Permission groups (empty) |
| `auth_permission` | 268 rows — all model-level permissions |
| `auth_group_permissions` | M2M (empty) |
| `django_content_type` | 67 rows — model registry |
| `django_migrations` | 132 rows — migration history |
| `django_session` | 1 373 rows — active sessions |
| `django_site` | 1 row — sites framework |
| `django_admin_log` | 11 rows — Django admin change log |

## Celery Beat Scheduler (all empty — not yet configured)

| Table | Purpose |
|---|---|
| `django_celery_beat_clockedschedule` | One-off clocked tasks |
| `django_celery_beat_crontabschedule` | Cron-style schedules |
| `django_celery_beat_intervalschedule` | Interval schedules |
| `django_celery_beat_periodictask` | Actual periodic tasks |
| `django_celery_beat_periodictasks` | Meta — last update timestamp |
| `django_celery_beat_solarschedule` | Solar event schedules |

---

## Query patterns

```sql
-- Recent admin actions on form submissions
SELECT al.created_at, u.email AS admin, al.action, al.note
FROM admin_panel_manageformsactionlog al
LEFT JOIN users_user u ON u.id = al.admin_user_id
ORDER BY al.created_at DESC LIMIT 20;

-- Social login users (have Google login)
SELECT u.email, sa.provider, sa.last_login
FROM socialaccount_socialaccount sa
JOIN users_user u ON u.id = sa.user_id
ORDER BY sa.last_login DESC;

-- Active sessions count
SELECT COUNT(*) FROM django_session WHERE expire_date > NOW();
```
