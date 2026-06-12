# Infodom DB — Users & Forms

## `users_user` — 66 rows
Custom Django user model (UUID primary key, email-based login).

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `email` | varchar(254) UNIQUE | NO |
| `first_name` | varchar(255) | NO |
| `last_name` | varchar(255) | NO |
| `password` | varchar(128) | NO |
| `is_superuser` | boolean | NO |
| `is_staff` | boolean | NO |
| `is_active` | boolean | NO |
| `date_joined` | timestamptz | NO |
| `last_login` | timestamptz | YES |

Standard Django M2M junction tables exist but are empty:
- `users_user_groups` — user ↔ auth_group
- `users_user_user_permissions` — user ↔ auth_permission

---

## `users_useraddressbinding` — 63 rows
Links one user to one verified address. Status drives the admin approval workflow.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `user_id` | uuid FK UNIQUE → `users_user` | NO | one binding per user |
| `address_id` | uuid FK UNIQUE → `address_info_address` | NO | one binding per address |
| `status` | varchar(1000) | NO | `PENDING` / `VERIFIED` / `REJECTED` |
| `created_at` | timestamptz | NO | |
| `proof_uploaded_at` | timestamptz | YES | when user uploaded document |
| `verified_at` | timestamptz | YES | when admin approved |
| `confirmation_method` | varchar(1000) | YES | how verified |
| `confirmation_document` | varchar(255) | YES | file path to uploaded proof |
| `confirmation_geolocation` | geometry (PostGIS) | YES | GPS coords at upload time |

---

## `form_category` — 6 rows
Top-level sections of the verification questionnaire.

| Column | Type |
|---|---|
| `id` | uuid PK |
| `name` | varchar(255) |

## `form_question` — 42 rows
Individual questions in the questionnaire.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `name` | varchar(255) UNIQUE | NO | question text / identifier |
| `points` | integer | NO | score weight for this question |
| `type` | varchar(10) | NO | `TEXT`, `BOOL`, `NUMBER`, `SELECT` |
| `data` | jsonb | NO | type-specific config (e.g. allowed options) |
| `order` | integer | NO | display order within category |
| `category_id` | uuid FK → `form_category` | NO | |

## `form_formsubmission` — 44 rows
One submission per (user, address) pair. `total_points` is the sum of points from all answered questions.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `user_id` | uuid FK → `users_user` | YES | composite UNIQUE with address_id |
| `address_id` | uuid FK → `address_info_address` | YES | composite UNIQUE with user_id |
| `status` | varchar(100) | NO | `PENDING` / `VERIFIED` / `REJECTED` |
| `submitted_at` | timestamptz | NO | |
| `verified_at` | timestamptz | YES | set by admin on approval |
| `total_points` | integer | NO | |

## `form_questionanswer` — 1.7 K rows
One row per (submission, question) pair.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `form_submission_id` | uuid FK | NO | composite UNIQUE with question_id |
| `question_id` | uuid FK → `form_question` | NO | composite UNIQUE |
| `answer_text` | text | NO | always present |
| `answer_number` | integer | YES | for NUMBER type questions |
| `answer_options` | jsonb | YES | for SELECT type questions |
| `optional_text` | text | NO | supplemental free-text |

---

## Query patterns

```sql
-- All pending form submissions
SELECT fs.id, u.email, a.full_address, fs.submitted_at, fs.total_points
FROM form_formsubmission fs
JOIN users_user u ON u.id = fs.user_id
JOIN address_info_address a ON a.id = fs.address_id
WHERE fs.status = 'PENDING'
ORDER BY fs.submitted_at;

-- Full answers for a submission
SELECT q.name AS question, q.type, qa.answer_text, qa.answer_number, qa.answer_options, fc.name AS category
FROM form_questionanswer qa
JOIN form_question q ON q.id = qa.question_id
JOIN form_category fc ON fc.id = q.category_id
WHERE qa.form_submission_id = '<submission_uuid>'
ORDER BY q.order;

-- Users without a verified address binding
SELECT u.id, u.email, u.date_joined
FROM users_user u
LEFT JOIN users_useraddressbinding b ON b.user_id = u.id AND b.status = 'VERIFIED'
WHERE b.id IS NULL AND u.is_staff = false;
```
