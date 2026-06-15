# Agent Instructions (Locumo Backend)

This repository contains the Django REST API backend for Locumo.
Frontend UI is handled by a separate React app.

## Scope and Architecture

- Build user-facing functionality as REST API endpoints (DRF).
- Do not build frontend UI in Django templates.
- Place code by domain:
  - `infodom/address_info/` for address/location logic
  - `infodom/payment/` for purchases/paywall/payments
  - `infodom/users/` for user/auth/account logic
  - `infodom/form/` for form workflows
  - `infodom/admin_panel/` for verification/backoffice actions
  - `infodom/utils/` for shared utilities/mixins

## API Conventions

- Prefer DRF ViewSets + serializers + permissions.
- Use `@action` for custom endpoints.
- Document custom endpoints/fields with drf-spectacular (`extend_schema`, etc.).
- When adding a new endpoint, or changing possible error paths on an existing endpoint, explicitly review error handling:
  - prefer the unified API error shape (`error.code`, `error.message`, `error.details`) for user-facing/API-facing failures
  - if the endpoint can return custom/business errors, add or update the corresponding drf-spectacular schema extension so OpenAPI documents the relevant error responses and error codes
  - if schema output changes intentionally, regenerate and commit the OpenAPI snapshot (`API.yaml`)
- Keep permission/auth behavior explicit, especially for feature-flagged endpoints.

## Data and Query Guidelines

- Optimize queryset per endpoint/action (avoid globally heavy prefetch/select unless needed).
- For large-result endpoints, consider indexes and query plans before adding app-level complexity.
- Normalize query params for API search/autocomplete when appropriate (e.g., trim whitespace).

## Testing Rules

- Every behavior change should include or update tests.
- Use pytest and existing test style in the touched app.
- Use factory classes for setup; add missing factories near existing ones.
- Use settings overrides/fixtures for feature flags and rate limits where needed.

## Commands (Canonical)

- Run full tests:
  - `make test`
  - or `docker compose run --rm django python -m pytest --create-db`
- Run focused tests:
  - `docker compose run --rm django python -m pytest <path> -k <expr> -q`
- Django checks:
  - `docker compose run --rm django python manage.py check`
- Lint/format hooks:
  - `pre-commit run --all-files`

## Migration Workflow

- If models/indexes/settings-driven schema changes are introduced:
  1. create migrations
  2. run migrations locally
  3. include migration files in the change
- Do not leave model changes without matching migrations.

## Git and Safety

- Never revert unrelated user changes.
- Never use destructive git commands (`reset --hard`, force checkout, etc.) unless explicitly requested.
- Keep diffs scoped to the requested task.

## Definition of Done

Before finishing a task:

1. Code changes are implemented and consistent with architecture above.
2. Tests for changed behavior are added/updated and passing (at least focused subset).
3. `manage.py check` and relevant lint/test commands pass or known blockers are explicitly reported.
4. API contract changes are documented (schema annotations and/or tests).
