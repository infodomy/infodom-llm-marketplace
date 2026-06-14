# Infodom DB — Payments & Reports

## How the payment flow works

```
User selects address to buy report for
  → payment_order created (billing details, address, status=PENDING)
  → payment_purchase created (links to order, payment provider session)
  → on success: payment_purchase.status updated, payment_userpurchasedreport created
  → user can now access that address's report
```

Free reports are earned by submitting the verification form and having it approved. Balance is tracked in `payment_userfreereports`; every credit/debit writes a row to `payment_freereportstransaction`.

---

## `payment_order` — 3 rows
Checkout order capturing billing details at purchase time.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `public_id` | varchar(32) UNIQUE | NO | shown to user |
| `status` | varchar(100) | NO | e.g. `PENDING`, `COMPLETED`, `CANCELLED` |
| `created_at` | timestamptz | NO | |
| `amount_in_pennies` | integer | NO | divide by 100 for PLN |
| `currency` | varchar(3) | NO | e.g. `PLN` |
| `customer_type` | varchar(3) | NO | `IND` (individual) / `BUS` (business) |
| `first_name` / `last_name` | varchar(150) | NO | |
| `email` | varchar(254) | NO | |
| `phone` | varchar(50) | NO | |
| `street` / `building_number` / `postal_code` / `city` / `country` | varchar | NO | billing address |
| `terms_accepted` | boolean | NO | |
| `waiver_14_days_accepted` | boolean | NO | GDPR 14-day withdrawal waiver |
| `needs_invoice` | boolean | NO | |
| `company_name` | varchar(255) | NO | for VAT invoice |
| `nip` | varchar(32) | NO | Polish tax ID (NIP) |
| `address_id` | uuid FK → `address_info_address` | YES | target address of the report |
| `user_id` | uuid FK → `users_user` | YES | |

## `payment_purchase` — 3 rows
Payment processor session (PayU, Stripe, etc.).

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `session_id` | uuid UNIQUE | NO | payment provider session id |
| `status` | varchar(200) | NO | provider status string |
| `payment_provider` | varchar(100) | NO | e.g. `payu`, `stripe` |
| `amount_in_pennies` | integer | NO | |
| `currency` | varchar(3) | NO | |
| `token` | varchar(255) | YES | provider token |
| `created_at` | timestamptz | NO | |
| `address_id` | uuid FK | YES | |
| `user_id` | uuid FK | YES | |
| `order_id` | uuid FK → `payment_order` | YES | |

## `payment_userpurchasedreport` — 30 rows
Access grant: user has paid for (or redeemed free credits for) a specific address report.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `user_id` | uuid FK → `users_user` | YES | composite UNIQUE with address_id |
| `address_id` | uuid FK → `address_info_address` | YES | composite UNIQUE with user_id |
| `purchase_id` | uuid FK → `payment_purchase` | YES | null if redeemed with free reports |
| `purchased_at` | timestamptz | NO | |

## `payment_userfreereports` — 59 rows
Current credit balance per user. `total_reports - total_used` = available free reports.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `user_id` | uuid FK UNIQUE → `users_user` | NO |
| `total_reports` | integer | NO |
| `total_used` | integer | NO |
| `last_updated` | timestamptz | NO |

## `payment_freereportstransaction` — 110 rows
Append-only ledger of free-report credit changes.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid PK | NO | |
| `user_id` | uuid FK → `users_user` | NO | |
| `transaction_type` | varchar(10) | NO | `CREDIT` / `DEBIT` |
| `reports` | integer | NO | positive delta (always positive, sign given by type) |
| `balance_after` | integer | NO | snapshot of balance at time of transaction |
| `created_at` | timestamptz | NO | |
| `form_submission_id` | uuid FK | YES | set for CREDIT earned from form approval |
| `unlocked_address_id` | uuid FK | YES | set for DEBIT spent to unlock an address |

## `payment_formsubmissionreportallocation` — 44 rows
Records how many free reports were (or will be) granted when a form submission is approved.

| Column | Type | Nullable |
|---|---|---|
| `id` | uuid PK | NO |
| `user_id` | uuid FK | NO |
| `form_submission_id` | uuid FK UNIQUE → `form_formsubmission` | NO |
| `total_reports` | integer | NO |
| `reports_granted` | integer | NO |
| `created_at` / `last_updated` | timestamptz | NO |

---

## Query patterns

```sql
-- Current free-report balance per user
SELECT u.email, ufr.total_reports - ufr.total_used AS available_credits
FROM payment_userfreereports ufr
JOIN users_user u ON u.id = ufr.user_id
ORDER BY available_credits DESC;

-- All addresses a user has access to (purchased or free)
SELECT a.full_address, upr.purchased_at,
       CASE WHEN upr.purchase_id IS NULL THEN 'free' ELSE 'paid' END AS access_type
FROM payment_userpurchasedreport upr
JOIN address_info_address a ON a.id = upr.address_id
WHERE upr.user_id = '<user_uuid>';

-- Revenue (completed orders only)
SELECT DATE_TRUNC('month', created_at) AS month,
       SUM(amount_in_pennies) / 100.0 AS revenue_pln,
       COUNT(*) AS orders
FROM payment_order
WHERE status = 'COMPLETED'
GROUP BY 1 ORDER BY 1;

-- Free-report ledger for a user
SELECT transaction_type, reports, balance_after, created_at
FROM payment_freereportstransaction
WHERE user_id = '<user_uuid>'
ORDER BY created_at;
```
