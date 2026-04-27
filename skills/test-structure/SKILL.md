---
name: test-structure
description: >-
  BDD-style test titles and feature-grouped describe blocks for Vitest and Cypress in
  ui-myfinance. Use when structuring or reviewing specs (tables, views, components),
  enforcing no root-level it(), or aligning feature-group coverage. For migrated HTML
  tables, pair with html-table-components Step 7 (test selector migration) and
  references/html-table-feature-groups.md.
---

# Test structure (ui-myfinance)

## Rules of thumb

- **BDD-style `it` titles** — describe outcome and condition (`when …, then …` where it helps).
- **Feature-grouped `describe` blocks** — no orphan `it()` at the root of a file; every test sits under a feature name.
- **Migrated `*HtmlTable.vue`** — use the canonical groups in **[references/html-table-feature-groups.md](references/html-table-feature-groups.md)** (Rendering, Sorting, Selection, …).

## Table migrations and tests

When `<mc-table>` is replaced by semantic `<table>`, specs must stop using **slot-based** cell queries (e.g. `` `[slot="${rowId}_delete"]` ``). That work is **mandatory in the same PR** as the SFC — see **[../html-table-components/SKILL.md](../html-table-components/SKILL.md#test-selector-migration-mandatory-in-the-same-pr-as-the-sfc)**.

## Related skills

- **[../html-table-components/SKILL.md](../html-table-components/SKILL.md)** — canonical table DOM, data attributes, and test selector migration.
- **[../migration-pr-review/SKILL.md](../migration-pr-review/SKILL.md)** — PR rubric including test selector hygiene.
