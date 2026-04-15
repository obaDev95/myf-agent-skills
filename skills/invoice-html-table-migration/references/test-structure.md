# Tests: BDD titles and feature-grouped `describe` blocks

Conventions for Vitest/Cypress specs around Vue components and views (including migrated `*HtmlTable.vue` and related screens).

## Mandatory rules

1. **BDD-style `it` titles** where practical: _when …, then …_ (or equivalent clear behavior wording).
2. **No root-level `it`** in component or view specs—every test sits under a **feature** `describe` (Sorting, Selection, Pagination, Page size, Rendering, Mobile expanded parity, etc.).
3. **Split capabilities**—do not put “select all” under **Sorting**, or mix unrelated behaviors in one `describe`.

## Feature groups (examples)

| Area               | Typical `describe` names                         |
| ------------------ | ------------------------------------------------ |
| Sorting            | Sorting                                          |
| Selection          | Selection, Row selection                         |
| Pagination         | Pagination                                       |
| Page size          | Page size                                        |
| Layout / snapshots | Rendering, Default layout                        |
| Mobile             | Mobile expanded parity, Mobile layout            |
| Conditional parity | When legacy used different markup per breakpoint |

## Component vs view (Vitest)

Same grouping applies to **views** that wrap a table (e.g. `EstatementTable.spec.ts`): use **Rendering**, **Pagination**, **Page size**, etc., not a vague lone `describe("snapshots")`.

## Cypress

Group commands under the same feature names as Vitest where possible so failures map to **functionality**, not file mechanics.

## Mobile conditionals and skill iteration

When a bug is “wrong thing visible on small viewport,” add or tighten tests that **toggle `appStore.isSmallScreen`** (or equivalent) and assert the **same user-visible fields** the legacy UI showed—then fold that rule into the **relevant feature skill** (e.g. table migration checklists) if your team maintains one.
