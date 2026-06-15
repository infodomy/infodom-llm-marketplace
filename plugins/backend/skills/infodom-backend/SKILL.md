---
name: infodom-backend
description: Guide backend development in the Infodom/Locumo Django REST API — architecture, DRF conventions, testing, migrations, and canonical commands. Use when working in infodom-backend, adding or changing API endpoints, or when the user asks how to develop backend features in this codebase.
---

# Infodom Backend Development Guide

Development guide for the **Locumo** Django REST API (`infodom-backend`).

Read the bundled reference before implementing or reviewing backend changes. When working inside the `infodom-backend` repository, prefer the live `AGENTS.md` at the repo root if it differs from the bundled copy.

## Reference file

| File | Read when the task involves… |
|---|---|
| `reference/AGENTS.md` | All backend work — architecture, API conventions, queries, testing, commands, migrations, git safety, definition of done |

## Output format

When guiding or reviewing backend work:

1. Confirm which reference section applies (architecture, API, testing, etc.).
2. State where code should live (domain app + likely files).
3. Call out API contract, migration, test, and OpenAPI impacts when relevant.
4. Finish with the definition-of-done checklist from `reference/AGENTS.md`.
