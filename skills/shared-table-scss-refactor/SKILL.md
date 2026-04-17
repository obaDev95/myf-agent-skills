---
name: shared-table-scss-refactor
description: >-
  Extend the shared table SCSS partials (`invoice-table.scss`, per-tab partials) without
  breaking sibling tables. Defines what belongs in the shared partial vs per-SFC scoped
  styles, the `@use` path contract, and the verification steps. Use whenever you catch
  duplicated CSS across `*HtmlTable.vue` files, add a new utility class consumed by more
  than one table, or touch `src/components/scoped-styles/invoice-table.scss`.
---

# Shared table SCSS refactor

This skill exists because the shipped `*HtmlTable.vue` files each duplicate ~100 lines of identical scoped CSS. Every duplicate is a future inconsistency: a fix that lands on one SFC and not its siblings is a regression by omission. The migration skills now mandate extraction; this skill is the playbook for doing it safely.

---

## Ownership map

In this repo, three tiers of SCSS concern each table:

| Tier | Where it lives | What belongs there |
|------|----------------|---------------------|
| **Shared partial** | `src/components/scoped-styles/invoice-table.scss` | Classes used by two or more invoice-domain tables: `column_right`, `data-dl`, `mobile-actions`, `show-link`, `line-height-20`, `align-right`, `adjust-width`, `sr-only`, cancelled-row contract (`cell--cancelled`, `cell--cancelled-status`, `disabled`, `cancelled-invoice-mobile`), sort-button chrome (`table-sort-button`, `table-sort-button__label`, `table-sort-button__icon`, `table-header-label`), row expander (`row-expander`, `row-expander__icon`), expanded-row chrome (`expanded-cell`, `desktop-notification-row`), BEM row-state selectors that follow `{tab}-table__row--hovered|selected|expanded`. |
| **Per-tab partial** | e.g. `src/components/credits/credits-table.scss` | Tab-specific layout that would distort sibling tables if promoted — credits-only `data-dl` widths, refund-reason paddings. Owned by the one tab that consumes it. |
| **SFC scoped styles** | `<style scoped lang="scss">` inside the `*HtmlTable.vue` | Classes that never appear outside this SFC (typically the `<section>` wrapper class and any truly one-off overrides). Should be a handful of lines, not a hundred. |

A class that is byte-identical across two or more tables is **not** scoped-styles material — it belongs in the shared partial. A class that is defined in the shared partial but consumed by only one table is also wrong: either the other tables should consume it, or it should move to a per-tab partial.

---

## `@use` path contract

Vite resolves `@use` relative to the `.vue` file's directory.

| SFC location | `@use` lines required |
|--------------|------------------------|
| `src/components/FooHtmlTable.vue` | `@use "scoped-styles/invoice-table";` |
| `src/components/credits/CreditsHtmlTable.vue` | `@use "./credits-table";` **and** `@use "../scoped-styles/invoice-table";` |

Rule: **any class used in the template that is defined in a partial this SFC did not load via `@use` will silently render unstyled.** This is the most common "works locally but broken in prod" bug during migration. Before merging, visually verify every cell that uses a shared utility.

When you move an SFC between folders, update the `@use` paths in the same commit. `npm run start` and `npm run build` surface missing paths ("Can't find stylesheet to import"); TypeScript does not.

---

## Extraction workflow

Run this workflow every time you catch duplicated CSS on a migration PR — not as a follow-up ticket.

### Step 1 — Identify duplication

```bash
# list every *HtmlTable.vue in scope
rg -l 'mds-table' src/components | rg 'HtmlTable.vue$'

# find candidate duplicated blocks by a recognisable selector
rg '\.row-expander|\.table-sort-button|\.expanded-cell|\.desktop-notification-row' -n src/components
```

A block that appears in ≥ 2 `*HtmlTable.vue` files is a candidate. A block defined differently in two SFCs is a **divergence** that must be reconciled before extraction.

### Step 2 — Reconcile differences

If two SFCs define the same class with different rules:

1. Diff them. Usually the difference is a missing modifier or a stale value, not intentional.
2. Agree on the correct shape with design / QA when the difference is visual.
3. Update all consumers to the agreed shape in a **separate commit** before extraction. Extraction is mechanical; reconciliation is not.

### Step 3 — Promote to the shared partial

1. Move the block into `src/components/scoped-styles/invoice-table.scss` under a named section comment (`// --- Sort button chrome ---`, etc.) so future readers see the grouping.
2. Remove the block from every consumer SFC's `<style scoped>`.
3. Verify each consumer SFC has `@use "scoped-styles/invoice-table";` (or the correct relative path for nested folders). Add it if missing.
4. Run `npm run build`. A missing `@use` surfaces here.
5. Load every consumer table in the browser at both breakpoints. Visual regressions from extraction are the most common post-merge surprise.

### Step 4 — Do not modify existing classes in place

If the extracted class already exists in `invoice-table.scss` and you need a different behaviour for one table, **add a modifier class** — do not mutate the base rule. Other tables depend on the current behaviour.

Acceptable:

```scss
.column_right { /* unchanged */ }
.column_right--tight { /* new variant */ }
```

Not acceptable: changing `.column_right`'s `align-items` and breaking three other tables that relied on the previous value.

---

## Splitting rules — when a partial grows too large

When `invoice-table.scss` exceeds ~200 lines or covers unrelated concerns (row chrome, cancelled-row contract, mobile action bars), split it by concern into additional partials forwarded by a barrel:

```scss
// src/components/scoped-styles/invoice-table.scss (barrel)
@forward "./table-row-state";
@forward "./table-sort-ui";
@forward "./table-mobile-expand";
@forward "./table-cancelled-row";
```

Consumers keep their single `@use "scoped-styles/invoice-table";` line. The barrel keeps the `@use` contract stable during refactors.

Do not split until the file meaningfully exceeds one screen. Premature splitting makes it harder to find the class you want.

---

## Scoped-styles rules inside each SFC

After extraction, the `<style scoped lang="scss">` in an `*HtmlTable.vue` should contain:

- `@use` lines for every partial whose classes appear in the template.
- The `<section>` wrapper class (e.g. `.paid-invoices-table { overflow: auto; }`).
- Nothing else by default.

Explicitly not allowed:

- `:deep()` on native table markup (`<table>`, `<thead>`, `<tbody>`, `<th>`, `<td>`, `.mds-table__…`) — these are same-SFC elements; scoped CSS applies to them without `:deep`.
- Duplicated copies of sort-button, row-expander, expanded-cell, desktop-notification-row, or row-state styles. Those are partial material.
- Hard-coded pixel paddings on `thead th` / `tbody td` — they belong in the shared partial so every table paginates and aligns the same way.

---

## Verification checklist

Before merging any change that touches a shared partial:

- [ ] `npm run build` passes.
- [ ] Every consumer SFC loads the partial via `@use` with the correct relative path.
- [ ] Each consumer table renders at both breakpoints with no visual diff (screenshot diff if the team supports it, eyeball verification otherwise).
- [ ] Component specs for every consumer table pass; updates to tests, if any, are in the same PR.
- [ ] No `*HtmlTable.vue` still defines a class that also lives in the partial.
- [ ] No class in the partial is consumed by only one table (move to per-tab partial) or zero tables (delete).

---

## Anti-patterns

- Fixing a visual bug by editing only the tab where it was reported, when the same class is duplicated in three siblings. Fix in the partial, not the SFC.
- Introducing a new shared class without `@use`ing it on every current consumer. The class silently no-ops on the SFCs that did not update.
- Mutating an existing shared class to change behaviour for one caller. Add a modifier instead.
- Promoting a one-off rule to the shared partial "in case other tables need it later". Shared means consumed by ≥ 2 tables *today*.
- Letting `<style scoped>` in a new `*HtmlTable.vue` copy the row-expander / sort-button CSS from a sibling instead of `@use`ing the partial. This is how the current duplication appeared.
