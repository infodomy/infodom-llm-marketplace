---
name: infodom-frontend
description: Guide frontend development in the Infodom/Locumo React monorepo — where code goes, component file layout, SCSS modules and the core design system, i18n, SWR data fetching, Redux modals, Formik + Zod forms, and coding standards. Use when working in infodom-frontend (apps/app, apps/landing, packages/core, packages/api), when writing or reviewing React/TypeScript components, or when the user asks how to develop frontend features in this codebase.
---

# Infodom Frontend Development Guide

Development guide for the **Locumo** React/TypeScript monorepo (`infodom-frontend`): `apps/app`,
`apps/landing`, `packages/core`, `packages/api`.

Read the bundled reference before implementing or reviewing frontend changes. When working inside
the `infodom-frontend` repository, prefer the live `AGENTS.md` at the repo root if it differs from
the bundled copy.

## Reference file

| File                  | Read when the task involves…                                                                                                                              |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reference/AGENTS.md` | All frontend work — file placement, component layout, styling, UI components, i18n, data fetching, Redux state, routing, forms, effects, coding standards |

## Before writing code

1. Decide where the code belongs (`components` vs `containers` vs `packages/core`) — core never imports from an app.
2. Check `@locumo/core/components` for an existing design-system component before writing a new one.
3. Confirm the conventions the change touches: SCSS modules + `color.get` / typography mixins (never literal hex or font sizes), `t()` from `useSafeT` for every user-facing string, `useDefaultSWR` + `useHttpClient` for data, Redux `openModal` for modals, Formik + Zod for forms.
4. For anything reactive, check [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) before reaching for `useEffect`.

## Output format

When guiding or reviewing frontend work:

1. Confirm which reference section applies (placement, styling, i18n, data, state, forms…).
2. State where the code should live (app/package + folder + `index.tsx` per component).
3. Call out design-system, i18n-key, generated-file (`Api.ts`, `*.module.scss.d.ts`, `generated-vars.scss`) and Redux-modal impacts when relevant.
4. Finish with the "Things to always do" checklist from `reference/AGENTS.md` — LF line endings, then `pnpm lint <changed files>`, `pnpm format`, `pnpm stylelint` (when SCSS changed) and `pnpm knip`.
