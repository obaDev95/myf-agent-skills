---
name: html-table-components
description: >-
  Builds and maintains semantic MDS HTML tables migrated from mc-table in ui-myfinance.
  Covers intentional data attributes (data-column-id on headers, data-cell-id on body cells),
  a11y, and test selectors. Use when adding or editing *HtmlTable.vue components,
  migrating tables from mc-table, or when the user mentions refunds selected table,
  data-column-id, data-cell-id, or HTML table migration in this repo.
---

# HTML table components (ui-myfinance)

## Related skills (restore if missing)

- **`invoice-html-table-migration`** (`skills/invoice-html-table-migration/SKILL.md`) is the **explicit mc-table → HTML table** playbook for this app: composables, reference components, test/e2e updates, and a migration checklist. Use its **Assessment rubric** when comparing a legacy `mc-table` paste to a migrated `*HtmlTable.vue`.
- **`mds-component-table-html`** (`skills/mds-component-table-html/`) documents **MDS foundations** for hand-written `<table>`: `.mds-table` modifiers, sticky headers, selection/expansion, a11y checklist, and a full Vue example.
- **Why it “vanished” on a new branch:** On many workflows, the `skills/` pack was introduced on branch `chore/add-mds-vue-skills` and is **not** part of `main`. Checking out a **new branch from `main`** therefore drops those files from the working tree unless they were committed on the branch you cut from. **Mitigation:** merge or cherry-pick `chore/add-mds-vue-skills`, or **commit** `skills/` on your feature branch so the pack travels with the repo.
- **Shared invoice table logic** in app code lives in `src/composables/InvoiceHtmlTable.composable.ts`, `InvoiceHtmlTablePresenters.composable.ts`, and table-specific composables (e.g. `OpenInvoicesTable.composable.ts`). The MDS skill does not replace those; it complements them.

## Scope

- Applies to **hand-written** `<table>` components in this repo (e.g. `*HtmlTable.vue`), not to legacy `mc-table` usage elsewhere.
- **Reference implementation:** `src/components/RefundsSelectedHtmlTable.vue`.
- Other migrated tables may still use the older pattern `data-header-id` on both `<th>` and `<td>`; **do not** rename them unless the team expands migration scope. For **new** work or when explicitly refreshing a component, prefer the intentional pattern below.

## Data attributes (intentional)

Avoid reusing **`data-header-id` on `<td>`**. It reads like a header identifier or a unique id, but historically it only meant “this cell belongs to column X,” and the same value repeated on every row.

For new or updated tables, use:

| Location | Attribute | Purpose |
|----------|-----------|---------|
| `<th>` | `data-column-id="<columnKey>"` | Stable **column** key (e.g. `invoiceNo`, `reference`). One header per column; value is not a globally unique DOM id. |
| `<td>` | `:data-cell-id="\`${row.id}-${columnKey}\`"` | **Unique per cell**: row id + column key. Pairs naturally with `tr` scoped by `:data-cy="row.id"` (or equivalent). |

Column keys should stay aligned with the conceptual column (and with tests), not with display strings.

## Semantics and layout

- Prefer `<section>` with `data-test` / `aria-label` for the table region when matching existing invoice/refund patterns.
- Use `scope="col"` on header cells.
- Keep **business logic** (formatting, emit payloads) in `<script setup>`; keep the template declarative.

### Scoped CSS and cell layout (migration hygiene)

- **Canonical `TableColumn` shape:** Types ship with **`@maersk-global/mds-components-core`** (re-export: `@maersk-global/mds-components-core/mc-table/types` → **`@maersk-global/mds-components-core-table`**). The **authoritative field list** is `TableColumn` in that package (e.g. `node_modules/@maersk-global/mds-components-core-table/src/lib/types.d.ts`). **Re-check after upgrading** `@maersk-global/mds-components-core` in case new optional fields appear.

- **Legacy `TableColumn` parity:** When migrating from `mc-table`, **walk every property** on each legacy column object and map it explicitly—**do not** assume only `width` or only fields used in one screen size matter.

- **Desktop vs mobile column arrays:** Many views define **`tableDesktopColumns`** and **`tableMobileColumns`** (or computed column sets). The **same `id`** can have **different** `width`, `align`, or other fields, or exist on only one breakpoint. Mirror that with **`v-if` / `v-show`** on `<th>` / `<td>`, **`@media`** in scoped CSS (align breakpoints with **`useAppStore().isSmallScreen`** / **`CONSTANTS.smallScreenBreakPoint`**), and separate footer padding if needed. Example: `RefundsSelectedHtmlTable.vue` — **200px** refund amount column on **desktop only**; **84px** actions on **all** sizes (matches legacy desktop + mobile configs).

- **Full `TableColumn` field → migration hint** (supplement the package typings; behavior comes from `mc-table` + your app):

  | Field | Plain HTML / Vue / CSS equivalent |
  |-------|-----------------------------------|
  | **`id`** | **`data-column-id`** on `<th>`; **`data-cell-id`** `${rowId}-${id}` on `<td>` — see **Data attributes (intentional)**. |
  | **`label`** | Header text in `<th>`; **`sr-only`** / `aria-label` when label was empty but the column remains (e.g. actions). |
  | **`align`** | **`.mds-table__cell--text-right`** / center / left utilities or `text-align` on `<th>` / `<td>`. |
  | **`verticalAlign`** | `vertical-align` on cells (often via scoped CSS), consistent with MDS row height. |
  | **`width`** | String (`100px`, `10%`) or **`{ min, max }`**: `<colgroup>` / `<col>`, or scoped rules on `th`/`td` by `data-column-id`. |
  | **`rowspan` / `colspan`** | Native **`rowspan`** / **`colspan`** on the relevant `<th>` / `<td>`. |
  | **`tabularFigures`** | **`.mds-tabular-figures`** on the table region or cell (match where legacy applied it). |
  | **`sortDisabled`** | Non-sortable: omit controls. Sortable table: per-column sort UI / **`aria-sort`** / composable logic only where sorting was allowed. |
  | **`sortdisableoninitialload`** | Initial sort state + “first column not sortable on load” in composable / props — mirror legacy `sortdefaultcolumnid` / table `sortdisabled` too. |
  | **`noWrap`** | **`white-space: nowrap`** (+ often **`min-width`**) on cell or **inner** wrapper. |
  | **`sorter`** | Custom compare in composable when not using default string/number sort. |
  | **`sticky`** | Sticky column: `position: sticky`, MDS scroll/sticky patterns — see **`mds-component-table-html`**. |
  | **`columns`** | **Grouped headers**: nested `<tr>` in `<thead>`, **`colspan`** / **`rowspan`** on `<th>`. |
  | **`footerColspan`** | `<tfoot>` / footer row **`colspan`** to match spanning behavior. |
  | **`renderAsHeader`** | Cell visually styled as header: use **`<th>`** (with correct **`scope`**) or preserve semantics per design. |
  | **`dataType`** | Formatting / locale in **`<script setup>`** (numbers, dates). |
  | **`description`** / **`descriptionWidth`** | Header **tooltip** / **`title`** / info icon + popover; constrain tooltip width if needed. |
  | **`subDataKey`** / **`subDataLabel`** | Secondary line in header or stacked label in `<th>`. |
  | **`rowClickDisabled`** | **`@click.stop`** on the cell (or wrapper) so row click does not fire for that column. |
  | **`headerTemplate`** / **`cellTemplate`** | Vue **slots** or inline template in the SFC replacing Lit templates. |
  | **`truncate`** | **`overflow: hidden`**, **`text-overflow: ellipsis`**, often with a fixed **`max-width`** / column width. |

  **Workflow:** Paste **both** desktop and mobile `TableColumn[]` (if any) next to the new markup and **tick off each field** per column per breakpoint.

- **Scope presentation rules to a named wrapper** on the region element (e.g. `<section class="…-table mds-table">`) and use **descendant selectors** for native `<table>` / `<thead>` / `<tbody>` / `<tr>` / `<th>` / `<td>` markup in the same SFC. That usually avoids reaching for **`:deep(.mds-table …)`** as the default: reserve `:deep` for styling **inside child components** or other cases where scoped CSS truly cannot see the target—not as a blanket substitute for a wrapper class on your own table tree.
- **Keep table cells on the table layout model.** `<th>` and `<td>` are expected to participate as **table cells**; changing them to **`display: flex`**, **`display: grid`**, or other non–table-cell displays can break **row alignment**, **shared baselines**, and **row chrome** (e.g. one column looks offset from its neighbors). If you need flex/grid for stacking or alignment **inside** a cell, apply it to a **wrapper** (`<div>`, `<span>`) **within** the cell, not on the cell itself.
- **Reuse MDS / app cell modifiers** for alignment and number styling (e.g. `.mds-table__cell--text-right`, `.mds-table__cell--number`) instead of forcing layout with flex on the `<td>`. When pulling in shared SCSS partials (such as `scoped-styles/invoice-table.scss`), confirm whether a utility is intended for **inner** content vs the **cell** before applying the class.

## Tests

- **Unit:** assert column visibility via `th[data-column-id='…']` when testing responsive column omission.
- **Component / Cypress:** target body cells with `td[data-cell-id="${row.id}-<columnKey>"]` inside `tr[data-cy="${row.id}"]` instead of repeating ambiguous column-only attributes on `<td>`.

## When migrating from mc-table

- **Column config audit:** For each legacy **`TableColumn`**, reconcile **all** fields from the **package typings** (not an ad hoc short list)—and reconcile **desktop vs mobile** column arrays when both exist. See **Legacy `TableColumn` parity** under **Scoped CSS and cell layout (migration hygiene)** above.
- Legacy tables identified columns via **column `id`** and **slot names** (e.g. row + column), not necessarily `data-header-id` in app markup.
- After migration, document any behavior parity (totals, delete rules, mobile-only blocks) in the component or PR; mirror important branches in unit/component tests.
- **Presentation props on `<mc-table>`** (e.g. disabling row hover) are component-internal; **do not** assume a fixed mapping to `.mds-table*` modifiers from skill text alone—see **`invoice-html-table-migration` → [mc-table-only props vs plain HTML table + MDS](../invoice-html-table-migration/SKILL.md#mc-table-only-props-vs-plain-html-table--mds-no-blind-parity-rule)**.

## Do not

- Bulk-find-replace `data-header-id` across the repo without an agreed migration ticket.
- Use `data-cell-id` without a stable row identifier—composite values must be deterministic for tests and automation.
