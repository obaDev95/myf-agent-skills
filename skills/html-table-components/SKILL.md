---
name: html-table-components
description: >-
  Migrate `<mc-table>` to semantic HTML `<table>` with `.mds-table` in ui-myfinance.
  Defines the canonical SFC shape, logic-preservation rules, shared SCSS/sub-component
  contract, data-attribute policy, and TableColumn parity. Repo-specific file names and
  migration commits live in references/ui-myfinance-tables.md. Use for any `*HtmlTable.vue`,
  every `<mc-table>` → `<table class="mds-table">` migration, and any table DOM/SCSS work.
---

# HTML table components (ui-myfinance)

## Intent

Each migration has three non-negotiable outcomes. Order matters; they are listed by precedence.

1. **Logic preserved.** Sorting, selection, expansion, pagination, hover menus, copy/analytics, store writes, cancelled/disabled row rules, API refetch, and virtualization must produce the same user-visible outcomes as the legacy `<mc-table>`. If a behavior is dropped, it is an intentional, documented change — not silent drift.
2. **Refactored for readability and maintainability.** Shared behavior lives in composables, shared markup in sub-components, shared styles in SCSS partials. A second migration that repeats the same block of markup or CSS as a sibling table is a bug — extract, do not duplicate.
3. **Visual parity through shared foundations.** Pixel-level equivalence with `<mc-table>`'s shadow DOM is **not** the goal; equivalent, stable presentation driven by `@maersk-global/mds-foundations` + the shared `invoice-table.scss` partial **is**. When presentation diverges, design signs off.

**A fix belongs in the shared layer.** When a regression is found on one table, the fix must land in the composable, partial, or sub-component that all similar tables consume — never as an inline patch on a single SFC.

---

## Repository-specific lookup

File names, legacy paths, and migration commit anchors for **this repo** are in **[references/ui-myfinance-tables.md](references/ui-myfinance-tables.md)**. Load that when you need exact filenames or `git show <commit>^:<path>` baselines; do not hard-code those names into portable advice.

---

## Migration pipeline (follow in order)

### Step 1 — Baseline the legacy table

1. Identify the legacy SFC: a checked-in `*Table.vue` containing `<mc-table>`, an inline `<mc-table>` in a view, or the pre-migration file at `git show <migration_commit>^:<path>` (anchors in the reference doc).
2. Record, verbatim:
   - Every `<mc-table>` attribute (`sortmanual`, `expand`, `expandopened`, `select`, `selectrowdisabled`, `height`, `headersticky`, `customstyles`, `disablerowhighlightonhover`, …).
   - The full `columns` / `mobileColumns` config arrays (`id`, `label`, `align`, `width`, `sortDisabled`, `noWrap`, `renderAsHeader`, `dataType`, etc.).
   - Every named slot (`` `${row.id}_${columnId}` ``, `` `${row.id}_expanded` ``, headers, footer) and its child components with **all** props.
   - Every conditional: `v-if` on columns/slots, `!appStore.isSmallScreen` / `isSmallScreen` branches, inline mobile-only fields embedded inside desktop columns.
   - Side-effects: shadow-DOM listeners, `manualSyncTableSelection`, imperative `tableRef` access, `customStyles` that inject per-row CSS.

### Step 2 — Build a parity matrix before writing markup

For every column × breakpoint, write down:

| Field (from `TableColumn`) | Target (HTML / Vue / CSS) |
|----------------------------|---------------------------|
| `id` | Column key used in `data-column-id` / `data-cell-id` and sort wiring |
| `label` | `<th>` text; `label: ""` → empty `<th>` with `sr-only` description when icon-only |
| `align` | `.mds-table__cell--text-right` / `--text-center` or scoped CSS |
| `verticalAlign` | Scoped `vertical-align` |
| `width` | `<col>` or scoped `width` / `min-width` / `max-width` |
| `rowspan` / `colspan` | Native attributes |
| `tabularFigures` | `mds-tabular-figures` on region or cell |
| `sortDisabled` / `sortdisableoninitialload` | `sortableColumns` set + composable gate |
| `noWrap` | `white-space: nowrap` (+ `min-width`) on inner wrapper |
| `sorter` | Custom compare in composable |
| `sticky` | `mds-component-table-html` sticky tokens |
| `columns` (grouped) | Additional `<thead>` rows with `colspan` / `rowspan` |
| `footerColspan` | `<tfoot>` colspan |
| `renderAsHeader` | `<th>` with correct `scope` |
| `dataType` / `description` / `subDataKey` / `subDataLabel` / `rowClickDisabled` / `headerTemplate` / `cellTemplate` / `truncate` | Script formatters, tooltips, `@click.stop`, slots, truncation CSS |

For every slot, list every prop the child cell receives. Dropping a prop (for example `DocumentReferenceCell :HBL="row.documentReference"`) is a regression, not a cleanup.

For every behavior — sorting, selection, expansion, pagination, analytics (`copyButtonClicked`), row/cancelled/disabled rules — note where it will live post-migration (composable, store action, presenter).

### Step 3 — Emit the canonical HTML structure

Every migrated table matches the same shape. Do not invent a new wrapper or reorder the standard columns.

```html
<section
  class="<tab>-table mds-table mds-tabular-figures"
  :class="tableClasses"
  :aria-label="tableRegionLabel"
  data-test="table"
>
  <table>
    <thead>
      <tr>
        <th v-if="showSelectionColumn" class="mds-table__column--row-selector" scope="col">…</th>
        <th v-if="appStore.isSmallScreen" class="mds-table__column--row-expander" scope="col">
          <span class="sr-only">Toggle row details</span>
        </th>
        <th v-for="col in …" scope="col" :data-column-id="col.id" …>…</th>
      </tr>
    </thead>
    <tbody>
      <template v-for="row in pagedRows" :key="row.id">
        <tr :data-cy="row.id" :class="rowClass(row.id)" …>
          <td v-if="showSelectionColumn" class="mds-table__column--row-selector" …>…</td>
          <td v-if="appStore.isSmallScreen" class="mds-table__column--row-expander" …>…</td>
          <td :data-cell-id="`${row.id}-<columnKey>`" …>…</td>
          …
        </tr>
        <tr v-if="appStore.isSmallScreen && isMobileRowExpanded(row.id)" class="mds-table__expanded-row" …>
          <td :colspan="mobileExpandedColspan" class="expanded-cell">…</td>
        </tr>
        <tr v-if="!appStore.isSmallScreen && expandedRowId === row.id" class="desktop-notification-row" …>
          <td :colspan="desktopColspan">…</td>
        </tr>
      </template>
    </tbody>
  </table>
</section>
```

Non-negotiable rules for this shape:

- **`<section>`** carries `mds-table` (and `mds-tabular-figures` when numeric columns dominate). It also carries `aria-label` / `data-test`. Responsive and scroll modifiers live in `tableClasses`.
- **`<th>` and `<td>` remain table cells.** Never apply `display: flex` or `display: grid` to them. Flex/grid stacks go on an **inner wrapper** (typically `.column_right`, `.data-dl`, `.mobile-actions` from the shared partial).
- **Header/body `v-if` must be paired.** Every `<th v-if="X">` has a matching `<td v-if="X">`. Breakpoint conditions must match the legacy template exactly (`!appStore.isSmallScreen` / `appStore.isSmallScreen`). Header/cell visibility can only diverge when the legacy table embedded a mobile-only value inside an existing column — document that case explicitly with a comment.
- **Row key is `:key="row.id"`.** Test hook is `:data-cy="row.id"` on the `<tr>`.
- **Colspans are computed per breakpoint, not hard-coded.** A single `desktopColspan` that spans a `<tr>` on both breakpoints is a bug whenever the column count differs by breakpoint — virtual-padding spacers, notification rows, and expanded-cell rows all risk over- or under-spanning when columns are gated by `v-if="!appStore.isSmallScreen"`. Count the `<th>` that actually render at the target breakpoint (selection column + mobile expander column + data columns with truthy `v-if`). When the desktop and mobile column counts differ, expose **two** computed spans (`desktopColspan`, `mobileColspan`) or a single `currentColspan` keyed off `appStore.isSmallScreen`. `mobileExpandedColspan` comes from `useInvoiceHtmlTableSelectionAndRows` and already handles the mobile case; any other colspan the SFC owns needs the breakpoint check.
- **Selection column is first, mobile expander column is second.** Every invoice-domain table follows this order so tests and SCSS `:first-child` selectors stay predictable.
- **The two expanded-row kinds are mutually exclusive.** The mobile `mds-table__expanded-row` renders on small screens; the `desktop-notification-row` renders on desktop when an error is surfaced. Tables that do not surface a desktop error (for example Disputed) simply omit that second `<tr>`.

### Step 4 — Wire logic through composables, not inlined state

Any behavior that is shared across the invoice-domain tables must come from the shared composables. Reinventing selection/hover/expand state in the SFC is a regression, regardless of whether it "works".

| Concern | Source |
|---------|--------|
| Row selection, hover, mobile expand, actionable invoice, row CSS classes | `useInvoiceHtmlTableSelectionAndRows({ pageRows, onSelectRows, tableVariant })` in `src/composables/InvoiceHtmlTable.composable.ts` |
| Sort UI (`aria-sort`, sort button icon, toggle) | `useInvoiceHtmlTableSortHelpers({ sortConfig, sortableColumns, sortChange })` (same file) |
| Mobile/expanded data rows | `useInvoiceHtmlTablePresenters(variant).expandedData(row)` in `InvoiceHtmlTablePresenters.composable.ts` |

For invoice-tab variants (Open/Paid/Credits/Disputed) and the tab-specific stores and per-tab quirks, see **[invoice-html-table-migration/SKILL.md](../invoice-html-table-migration/SKILL.md)**.

No imperative `tableRef.shadowRoot.querySelectorAll(...)` code is allowed after migration. All hover/selection/expansion is declarative (`@mouseenter`, `@mouseleave`, `:class`, `v-if`) and lives in the composable.

### Step 5 — Shared SCSS contract (no duplication)

The scoped styles of every migrated HtmlTable must satisfy these rules:

1. **Any class used in the template that is defined in a shared SCSS partial must be `@use`'d in this SFC's `<style scoped lang="scss">`.** Vite resolves `@use` paths relative to the `.vue` file's directory. Nested folders add `../`. A template that uses `column_right`, `data-dl`, or `mobile-actions` without the corresponding `@use "scoped-styles/invoice-table"` (or relative path) will silently render unstyled.
2. **No `:deep()` on native table markup in this SFC.** `<table>`, `<thead>`, `<tbody>`, `<th>`, `<td>`, `.mds-table__column--row-expander`, `.mds-table__column--row-selector`, `.mds-table__expanded-row` all live inside the same SFC as the styles; Vue's scoped transform already applies. `:deep()` is reserved for piercing into **another** component's template (rare here).
3. **No inline `style=""` attributes for layout.** Widths, paddings, alignments are classes — either from `invoice-table.scss`, a local partial next to the SFC, or scoped CSS. The only acceptable inline style is the dynamic `tableInlineStyles` computed for explicit `height` props.
4. **Do not duplicate cross-SFC CSS.** Any block that appears byte-identical in a sibling `*HtmlTable.vue` must be extracted. The current home for shared table chrome (sort buttons, row expander, hovered/selected row, expanded cell, notification row, `sr-only`, header/cell paddings) is **`src/components/scoped-styles/invoice-table.scss`** — extend it and remove the duplicates when you touch a file.
5. **BEM-style row state classes.** Row hovered/selected/expanded classes follow `{tab}-table__row--hovered|selected|expanded`, generated by `useInvoiceHtmlTableSelectionAndRows` via the `tableVariant`. Do not hand-roll alternatives.

### Step 6 — Data-attribute policy (one convention per table)

Two conventions exist. Each table picks **one** and sticks to it.

| Location | Convention A (legacy — used by all invoice-tab tables today) | Convention B (refreshed — estatement, refunds-selected) |
|----------|--------------------------------------------------------------|--------------------------------------------------------|
| `<th>`   | `data-header-id="<columnKey>"`                                | `data-column-id="<columnKey>"`                          |
| `<td>`   | `data-header-id="<columnKey>"` (repeats per column)          | `:data-cell-id="\`${row.id}-<columnKey>\`"` (unique per cell) |

Additional, always required:

- `<tr>` carries `:data-cy="row.id"`; this is how tests disambiguate rows regardless of convention.
- `<section>` carries `data-test="table"` (invoice tabs) or a tab-specific variant (`data-test="estatement-table"` etc.) so component specs can target the region.
- Sort buttons carry `data-test="sort-button-<columnKey>"`; select-all carries `data-test="select-all-invoices"`; row expander carries `data-test="row-expander"`. These are consumed by Cypress specs.

**Switching conventions is a ticketed change, not a drive-by.** Moving a shipped invoice tab from A to B requires grepping `tests/unit`, `tests/component`, and `tests/e2e` for `data-header-id` and updating step definitions in the same PR. Do not mix conventions inside one SFC.

`data-header-id` on `<td>` is **not unique per cell** — the same value repeats for every row in a column. Any new test that needs a single cell hook must either scope through `tr[data-cy="<id>"]` or adopt Convention B.

### Step 7 — Tests

- Unit (Vitest) and component (Cypress) tests live per `*HtmlTable` and follow **[test-structure](../test-structure/SKILL.md)**: BDD `it(...)` titles, feature-grouped `describe` blocks (Sorting, Selection, Pagination, Mobile expanded parity, …), no root-level `it`.
- Toggle `appStore.isSmallScreen` via Pinia to cover responsive branches.
- When the table virtualizes rows (Open), do not assume every `pagedRows` item is in the DOM at once — scroll the container, widen the viewport, or stub the virtualizer.
- When the table refetches on sort or page change (Disputed), assert the store actions fired, not only the column state.
- When the table has cancelled/disabled rows (Paid), assert both the `cell--cancelled` class and the absence of interactive handlers.

### Step 8 — Verify

1. Run unit and component specs for the touched HtmlTable and parent view.
2. Run relevant e2e if selectors or step definitions changed.
3. Diff the legacy template's mobile/expanded branches against the new presenter output **and** any inline mobile blocks — data shown only in legacy mobile/expanded slots must still surface somewhere in the new template.
4. Open the page at the two breakpoints and confirm the same rows, amounts, currency, totals, empty/loading/error states, and analytics events fire.
5. Document every intentional UX or DOM change in the PR.

---

## `mc-table` → HTML feature map

| Legacy `<mc-table>` | Post-migration |
|---------------------|----------------|
| `sortmanual` + `@sortchange` | `useInvoiceHtmlTableSortHelpers` + `<button>` in `<th>` with `aria-sort` |
| `select` + `@selectchange` | `showSelectionColumn` `<th>`/`<td>` with `<mc-checkbox>` + `toggleSelectAll` / `toggleRowSelection` from `useInvoiceHtmlTableSelectionAndRows` |
| `expand` + `expandopened` + `${row.id}_expanded` slot | `<tr class="mds-table__expanded-row">` (mobile) and `<tr class="desktop-notification-row">` (desktop error) driven by `isMobileRowExpanded` / `expandedRowId` |
| `selectrowdisabled` (e.g. cancelled rows) | `cell--cancelled` class on each `<td>` + `pointer-events: none` via the shared partial; ignore the selection change in the parent handler |
| `customstyles` string injected into shadow DOM | Scoped SCSS on this SFC (for SFC-local tweaks) or the shared `invoice-table.scss` partial (for anything reusable) |
| `disablerowhighlightonhover` | `mds-table--disable-row-highlight-on-hover` on the `section.mds-table` wrapper **only when** legacy had the prop — do not add it by default |
| Shadow-DOM `tableRef.shadowRoot.addEventListener(...)` for hover | `@mouseenter` / `@mouseleave` on `<tr>` calling `handleRowMouseEnter` / `handleRowMouseLeave` |
| `manualSyncTableSelection(tableRef)` on filter/page change | Automatic via composable's `watch(pageRows)` resetting expansion + reactive `selectedInvoiceIds` |
| `height` / `headersticky` props | `section` inline `maxHeight` + `mds-table--scrollable` / `mds-table--header-sticky` classes |
| Named slot `${row.id}_<columnKey>` | Explicit `<td>` whose content replicates the slot children (keep every prop on leaf components) |

---

## Canonical sort-button shape

Repeat this per sortable `<th>`; the shape comes from `useInvoiceHtmlTableSortHelpers` and must look the same on every table. When a migration ships the fifth copy of this block, extract it to `SortableColumnHeader.vue` and delete the duplicates — treat further duplication as debt, not a style choice.

```html
<th
  scope="col"
  :aria-sort="getAriaSort('<columnKey>')"
  :data-column-id|:data-header-id="<columnKey>"
>
  <button
    v-if="isColumnSortable('<columnKey>')"
    type="button"
    :class="['table-sort-button', sortButtonClass('<columnKey>')]"
    :aria-label="`Toggle sort for ${$t('<heading.i18n>')}`"
    :data-test="`sort-button-<columnKey>`"
    @click="toggleSort('<columnKey>')"
  >
    <span class="table-sort-button__label">{{ $t("<heading.i18n>") }}</span>
    <mc-icon
      :icon="sortIconName('<columnKey>')"
      class="table-sort-button__icon"
      :class="{ 'is-active': isSortedColumn('<columnKey>') }"
      aria-hidden="true"
    />
  </button>
  <span v-else class="table-header-label">{{ $t("<heading.i18n>") }}</span>
</th>
```

The `v-if="isColumnSortable(...)"` branch + `<span v-else>` fallback is mandatory — it is the safe pattern when `sortableColumns` is a prop, a config, or changes by breakpoint.

---

## Canonical mobile row-expander shape

```html
<td v-if="appStore.isSmallScreen" class="mds-table__column--row-expander" :data-cell-id|:data-header-id="'row-expander'">
  <button
    type="button"
    class="row-expander"
    data-test="row-expander"
    :aria-expanded="isMobileRowExpanded(row.id)"
    :aria-controls="`expanded_${row.id}`"
    @click="toggleMobileRow(row.id)"
  >
    <mc-icon
      icon="chevron-down"
      class="row-expander__icon"
      :class="{ 'is-expanded': isMobileRowExpanded(row.id) }"
      aria-hidden="true"
    />
    <span class="sr-only">
      {{ isMobileRowExpanded(row.id) ? "Hide <entity> details" : "Show <entity> details" }}
    </span>
  </button>
</td>
```

The `.row-expander` / `.row-expander__icon` styles live in the shared partial, not the SFC.

---

## Canonical mobile-expanded and desktop-error rows

```html
<tr
  v-if="appStore.isSmallScreen && isMobileRowExpanded(row.id)"
  :id="`expanded_${row.id}`"
  class="mds-table__expanded-row"
  :data-cy="`expanded-${row.id}`"
>
  <td :colspan="mobileExpandedColspan" class="expanded-cell" data-test="expanded-row-content">
    <div v-for="(item, key) in expandedData(row)" :key="`data-dl-row-${key}`" class="data-dl">
      <div class="data-dl__dt">{{ $t("tableHeadings." + item.key) }}</div>
      <div class="data-dl__dd">{{ item.text }}</div>
    </div>
    <div class="mobile-actions">
      <ActionButtons :selectedInvoices="actionableInvoice(row.id)" displayed-from="ResponsiveTable" />
    </div>
  </td>
</tr>
<tr
  v-if="!appStore.isSmallScreen && expandedRowId === row.id"
  :id="`expanded_${row.id}`"
  class="desktop-notification-row"
  :data-cy="`expanded-${row.id}`"
>
  <td :colspan="desktopColspan">
    <mc-notification
      icon="exclamation-octagon"
      appearance="error"
      :body="$t('stickyPanel.noDocumentMessage')"
      :closable="true"
      @close="onCloseError()"
    />
  </td>
</tr>
```

- `expandedData(row)` is always a presenter result — do not compute it in the SFC. When the legacy mobile/expanded slot rendered fields that the presenter does not yet return, **add them to the presenter** (in `InvoiceHtmlTablePresenters.composable.ts`); do not inline a one-off list in the SFC.
- Some domains legitimately render **inline** mobile fields inside a desktop column (for example a status tag inside an amount cell on small screens). Keep that pattern when the legacy template had it, with a comment pointing at the legacy branch; do not duplicate the same value in both places.
- The desktop error row is omitted on tables whose legacy did not expose an expanded desktop slot (for example Disputed).

---

## Stacked numeric cells (amount + secondary line)

Right-aligned amount columns use the shared `column_right` flex stack. Never replace it with inline flex or put flex on the `<td>` itself.

```html
<td class="mds-table__cell--number mds-table__cell--text-right" …>
  <div class="column_right">
    <strong>
      {{ row.currency }}<span style="padding-left: 4px" class="mds-tabular-figures">{{
        formatCurrency(row.amount, row.currency)
      }}</span>
    </strong>
    <SubCell v-if="row.secondaryAmount" :value="…" tooltip="toolTip.invoiceAmount" />
    <template v-if="appStore.isSmallScreen">
      <!-- inline mobile-only tags or copy go inside the same stack -->
    </template>
  </div>
</td>
```

The `column_right` class lives in `src/components/scoped-styles/invoice-table.scss`. The SFC **must** `@use "scoped-styles/invoice-table"` (path relative to the SFC) for the stack to render correctly. This is the single most common source of silent visual regressions in this migration.

---

## Wrapper and responsive classes

`tableClasses` is always a computed map. Entries that every tab ships:

- `mds-table--scrollable` — always true for invoice tabs. Desktop overflow plus sticky header rely on it.
- `<tab>-table--mobile` — true on small screens. Carries the only responsive CSS specific to the tab.
- `mds-table--header-sticky` — true when the parent view asks for it (Open accepts a `headerSticky` prop).
- `mds-table--disable-row-highlight-on-hover` — **only** when the legacy `<mc-table>` carried `disablerowhighlightonhover`. Do not add by default.

`tableInlineStyles` is used only when the SFC accepts a `height` prop and sets `maxHeight` / `height` on the `<section>`. Anything else must be a class.

---

## Do / Don't summary

**Do**

- Match the canonical `<section>` → `<table>` → `<thead>` / `<tbody>` shape exactly.
- Wire selection, hover, expansion, sort, and row classes through the shared composables.
- Keep every slot's child-component props — `DocumentReferenceCell :BL`, `:HBL`, `:customer-reference`, `copy-field`, `copy-page`, prefix labels all transfer verbatim unless a ticket says otherwise.
- `@use` every shared partial whose classes you reference in the template.
- Pair `<th v-if>` and `<td v-if>` on identical conditions.
- Pick one data-attribute convention per table; document convention switches with test updates in the same PR.
- Compute colspans from `showSelectionColumn` + visible columns — never hard-code.
- Keep row keys (`:key="row.id"`) and row test ids (`:data-cy="row.id"`) stable.
- Extract shared markup/styles to partials or sub-components the moment duplication appears.

**Don't**

- Do not apply `display: flex` or `display: grid` to `<th>` / `<td>`; use an inner wrapper.
- Do not write `:deep(.mds-table …)` for markup that lives in the same SFC.
- Do not inline a second copy of the sort button, row expander, expanded row, or scoped CSS when a sibling SFC already ships it — extend the shared layer instead.
- Do not attach imperative listeners to `tableRef.shadowRoot`. The HTML table has no shadow root.
- Do not map legacy `customstyles` strings one-for-one to scoped CSS — re-express the UX with classes and composable row state.
- Do not assume another tab's `expandedData` applies to a new tab — always compare the legacy mobile/expanded slot against the presenter output for **that** variant.
- Do not rename `data-header-id` on shipped invoice tables without a ticket and coordinated test updates.

---

## Related skills

- **[invoice-html-table-migration](../invoice-html-table-migration/SKILL.md)** — composables, tab-specific stores, Open virtualization, Paid cancelled rows, Disputed API sort, Credits presenter caveats, Estatement column-order handling.
- **[mds-component-table-html](../mds-component-table-html/SKILL.md)** — `.mds-table` modifier catalogue, sticky/scroll, a11y.
- **[test-structure](../test-structure/SKILL.md)** — BDD spec titles and feature-grouped describes.
