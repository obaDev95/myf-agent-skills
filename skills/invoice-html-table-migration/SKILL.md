---
name: invoice-html-table-migration
description: >-
  Migrate invoice-domain tables (Open, Paid, Credits, Disputed, Estatement) from
  `<mc-table>` to semantic HTML `<table>` with `.mds-table` in ui-myfinance. Covers the
  invoice composables, presenter contract, per-variant wiring (virtualization, cancelled
  rows, API sort, precomputed formatted data), and test expectations. Use when migrating
  or reviewing any invoice-tab or estatement HtmlTable.
---

# Invoice tables: `<mc-table>` → HTML `<table>`

## How to use this skill

1. Read **[html-table-components/SKILL.md](../html-table-components/SKILL.md)** first — it defines the pipeline, canonical structure, shared SCSS contract, data-attribute policy, and `TableColumn` parity matrix that every migration must satisfy.
2. Use **this** file for invoice-domain specifics: which composables wire which behavior, how each tab differs, and where the presenter data comes from.
3. Use **[html-table-components/references/ui-myfinance-tables.md](../html-table-components/references/ui-myfinance-tables.md)** for legacy paths and `git show <commit>^:<path>` baselines.

No invoice migration is complete until it satisfies the three intents from the generic skill: **logic preserved**, **refactored for readability and maintainability**, **visual parity through shared foundations**. A fix for a bug found on one tab lands in the shared composable/partial/presenter, not as an inline patch on one SFC.

---

## Shared composables (`src/composables/InvoiceHtmlTable.composable.ts`)

### `useInvoiceHtmlTableSelectionAndRows<T>({ pageRows, onSelectRows, tableVariant })`

`tableVariant` is `"open" | "paid" | "credits" | "disputed" | "estatement"`. The composable reads `invoicesStore`, `appStore`, and the per-variant store it needs; the SFC does not reimplement any of this.

Returned state / handlers that the template must consume by name (not reinvent):

| Export | Consume in template as |
|--------|------------------------|
| `hoveredId` | `:data-state="hoveredId === row.id ? 'hovered' : undefined"` (optional, parity with legacy `data-state`) |
| `expandedRowId` | `v-if="!appStore.isSmallScreen && expandedRowId === row.id"` for the desktop error row |
| `pageSelectionState` | `:checked="pageSelectionState.allSelected"`, `:indeterminate="pageSelectionState.indeterminate"` on the header checkbox |
| `showSelectionColumn` | `v-if="showSelectionColumn"` on `<th>` and `<td>` for the row selector |
| `mobileExpandedColspan` | `:colspan="mobileExpandedColspan"` on the mobile expanded row's single `<td>` |
| `toggleSelectAll` / `toggleRowSelection(row)` | `@change` on header and row `<mc-checkbox>` |
| `toggleMobileRow(row.id)` / `isMobileRowExpanded(row.id)` | `@click` and `:aria-expanded` on the row expander button |
| `isRowSelected(row.id)` | `:checked` on the row checkbox |
| `rowClass(row.id)` | `:class` on the `<tr>` — emits the BEM `{tab}-table__row` + hovered/selected/expanded modifiers |
| `handleRowMouseEnter(row.id)` / `handleRowMouseLeave` | `@mouseenter` / `@mouseleave` on the `<tr>` |
| `actionableInvoice(row.id)` | prop to `<ActionButtons>` — returns the single invoice as an array or `[]` |
| `setActionableInvoice` / `clearActionableInvoice` | `defineExpose` them only on Paid, where the parent drives actionable invoice from outside the table |
| `onCloseError` | `@close` on the desktop notification row |

The composable also watches `pageRows` and `appStore.isSmallScreen` to reset expansion state. Do not duplicate those watches.

### `useInvoiceHtmlTableSortHelpers<TColumn>({ sortConfig, sortableColumns, sortChange })`

`sortableColumns` is a `Set<TColumn>`. `sortConfig` is a `ComputedRef<{ column?: TColumn; direction: Direction }>` sourced from `invoicesStore.<tab>TabSortConfig`. `sortChange` is the tab's bridge to store/API state.

Every sort button must use the exports `isColumnSortable`, `isSortedColumn`, `sortButtonClass`, `sortIconName`, `getAriaSort`, `toggleSort`. The canonical `<th>` sort-button shape in `html-table-components/SKILL.md` is the single template that all variants follow. The `v-if="isColumnSortable(...)"` + `<span v-else>` fallback is required when `sortableColumns` is dynamic (Open); it is optional when every sort-button column is statically sortable (Paid / Credits / Disputed). `data-test="sort-button-<key>"` is always required.

### `useInvoiceHtmlTablePresenters(variant).expandedData(row)`

`variant` is `"open" | "paid" | "credits" | "disputed"` (estatement has no expanded row). `expandedData(row)` returns `{ key, text }[]` — each item is rendered as one `data-dl__dt` / `data-dl__dd` pair inside the mobile expanded row.

**Presenter-first rule.** When the legacy mobile/expanded slot rendered a field that does not come out of `expandedData`, **add it to the presenter** in `InvoiceHtmlTablePresenters.composable.ts`; do not hand-roll a one-off `<div>` in the SFC. The only exceptions are:

- Fields that the legacy template intentionally rendered inline in a **desktop column on small screens** (for example the status tag inside Open's amount cell on mobile). These stay inline with a comment; duplicating them in `expandedData` would double-render on mobile.
- Cancelled-state copy that the legacy template rendered as a prefix to the expanded list (Paid's "cancelled invoice" header). That stays as an explicit `<div>` above the presenter loop.

### `useInvoiceSelection(pageRows)`

Lives in `src/composables/InvoiceSelection.composable.ts`. Wraps `invoicesStore.checkedInvoicesIds` writes and exposes `handleSelectionChange(selectedRows)`. **Paid** and **Credits** (and Estatement) use it to bridge between `toggleSelectAll` / `toggleRowSelection` and the store. **Disputed** writes `invoicesStore.checkedInvoicesIds` directly because its selection shape differs. **Open** uses `useOpenInvoicesTableContext`, which wraps `useInvoiceSelection` internally.

---

## Per-variant wiring

These are the only places the variants legitimately differ. Anything else should not differ; if it does, converge it in the shared layer.

### Open (`OpenInvoicesHtmlTable.vue`)

- **Context composable**: `useOpenInvoicesTableContext({ hideInvoicePdfLink })` provides `perPageCount`, stores, `pagedInvoices`, `totalPages`, `currentPage`, `paginationChange`, `onSelect`, `setResultsPerPage`, `sortChange`, `showInvoiceLink`, `getSecondaryCustomerRef`. The SFC should not reconstruct any of this.
- **Window virtualization**: `useWindowVirtualTable(CONTAINER_REF_NAME, { count: pagedInvoices.length })` returns `virtualRows` (`visibleInvoices`), `paddingTop`, `paddingBottom`. The `<tbody>` is iterated by index (`v-for="visibleInvoice in visibleInvoices"`, `pagedInvoices[visibleInvoice.index]`), wrapped in `<tr v-if>` / `<td :colspan :style>` virtual padding rows. Tests must not assume "row count in DOM = page size" — scroll the container, widen the viewport, or stub the virtualizer.
- **Sort gating**: Open is the only invoice tab that accepts an arbitrary `sortableColumns` set different from its column list, so `v-if="isColumnSortable('…')"` with a `<span v-else>` fallback is mandatory on every sortable `<th>`. Use the same pattern on the other tabs to keep the shape consistent even when their `sortableColumns` currently covers every header.
- **Tab cleanup**: `onUnmounted` resets `openInvoicesStore.resetDivisionFilter()` and `invoicesStore.REMOVE_CANCELLED_INVOICES()`.
- **Region label**: `$t("openinvoices.title")`. No `data-test="table"` on the wrapper (the outer container is refed as `scrollContainerRef` for virtualization) — add one if a component spec needs it and update tests in the same PR.

### Paid (`PaidInvoicesHtmlTable.vue`)

- **Selection bridge**: `useInvoiceSelection(pagedInvoices).handleSelectionChange`.
- **Cancelled rows**: `isCancelledPaymentInfo(row)` is the one authority. Every cell on a cancelled row receives `:class="{ 'cell--cancelled': isCancelledPaymentInfo(row) }"`. The status cell additionally gets `cell--cancelled-status` so `pointer-events: auto` keeps the status tooltip interactive. The mobile expanded row renders an explicit `<div class="cancelled-invoice-mobile" data-test="cancelled-invoice-mobile">` above the presenter loop. The `.cell--cancelled`, `.cell--cancelled-status`, `.disabled`, and `.cancelled-invoice-mobile` styles live in the shared partial — extend it there, do not reintroduce them per SFC.
- **Cancelled-row selection contract**: legacy Paid used `selectrowdisabled="isCancelledPaymentInfo"`, so `<mc-table>` excluded cancelled rows from **both** direct click and the header "select all". CSS `pointer-events: none` on `cell--cancelled` blocks direct clicks on a row's checkbox, but it does **not** block the header select-all — `toggleSelectAll` fires programmatically and writes `[...pageRows.value]` into the selection store regardless of cancelled state. Close the gap one of two ways:
  1. Extend `useInvoiceHtmlTableSelectionAndRows` with a `rowSelectable` predicate and filter inside `toggleSelectAll` / `toggleRowSelection`. **Preferred** — the contract stays in the composable and every consumer inherits it.
  2. Filter cancelled rows in the `onSelect` handler before forwarding to `handleSelectionChange`. Acceptable for Paid alone when a composable change is out of scope.
  Do not rely on CSS `pointer-events: none` as the sole guard — it is a visual lock, not a selection guard.
- **Status column tooltip**: cancelled tags render inside an `<mc-tooltip>` wrapper with `data-test="tooltip-cancelled-invoice"` so the hover message ("This invoice has been cancelled") stays visible.
- **Parent-driven actionable invoice**: `defineExpose({ setActionableInvoice, clearActionableInvoice })` so the parent view (Paid sticky panel) can set the actionable invoice from outside mouse events.
- **Pagination**: lives in the SFC because the parent view doesn't own it. Page changes call `invoicesStore.fetchDisputedStatus("paidInvoices")` on navigation and size changes.
- **Lifecycle**: `onMounted(() => paidInvoiceStore.resetDivisionFilter())`.
- **Region label**: `$t("paidInvoices.title")`.

### Credits (`credits/CreditsHtmlTable.vue`)

- **Precomputed row fields**: the SFC maps `creditInvoiceStore.pagedFilteredInvoices` into `(CreditInvoice & { formattedCredit, formattedDate })[]`. Keep formatting in the SFC-local `computed` when it is specific to this view's labels; when it becomes reusable, promote it to a helper in `src/lib/credit-invoices.helpers.ts` and thin the SFC.
- **Presenter + inline parity**: `useInvoiceHtmlTablePresenters("credits").expandedData` includes `reference`, `creditsDate`, and (if present) `invoiceType`. The SFC additionally renders `row.reason` **inline** in the credit-number cell on small screens and again inline in the `isRefundable` cell on desktop — this mirrors the legacy layout and is not a bug. Do not move `reason` into `expandedData` without design approval, because that would change the mobile layout.
- **Mobile type/status in amount cell**: on small screens, the `availableCredit` cell additionally renders `row.isRefundable` and the refund-status tag, because the dedicated `type` and `status` columns are desktop-only. This is the canonical example of "inline mobile value inside a desktop column" — every future table with the same legacy shape must follow this pattern, not add a synthetic mobile column.
- **Nested SCSS**: `<style scoped>` does `@use "./credits-table"` for credits-specific data-dl widths and `@use "../scoped-styles/invoice-table"` for the shared partial. When moving the SFC out of `src/components/credits/`, adjust both paths.
- **Expanded error row**: Credits, like Open and Paid, surfaces the desktop "no document" notification via `expandedRowId === row.id`. Disputed does not.
- **Region label**: `$t("common.creditsRefunds")`.
- **Sort**: `sortableColumns = { creditNumber, availableCredit, creditDate, isRefundable }` via `invoicesStore.changeSort("creditsTab", …)`.

### Disputed (`DisputedInvoicesHtmlTable.vue`)

- **Dual-write sort**: Disputed is the only tab where sort writes to **two** places.
  1. `invoicesStore.changeSort("disputedTab", column, direction)` keeps `disputedTabSortConfig` reactive for `useInvoiceHtmlTableSortHelpers`.
  2. `DISPUTED_UI_COLUMN_TO_API_SORT_BY` translates the UI column name to the API field, then `disputesStore.UPDATE_SORT_VALUE({ sort_by, id })` + `await disputesStore.getSortFilteredDisputedInvoiceList()` refetches the list. Reviewers must see both writes.
- **No desktop error row**: Disputed does not surface the "no document" notification, so the second `<tr v-if="!appStore.isSmallScreen && expandedRowId === row.id">` is intentionally omitted. Do not copy it in from Open/Paid/Credits during a migration refresh.
- **Row data source**: `computed(() => disputesStore.invoicesVM)` — pagination is driven by the store, not a local `pagedInvoices` slice.
- **Pagination**: `disputesStore.UPDATE_PAGE_NUMBER(page)` + `getSortFilteredDisputedInvoiceList()`; `setTotalPages` writes `appStore.resultsPerPage` and calls `paginationChange(1)`.
- **Selection**: writes directly to `invoicesStore.checkedInvoicesIds` via `onSelect` (no `useInvoiceSelection`).
- **Sort gating**: `sortableColumns = { ohpDisputeId, invoiceNo, reference, disputedAmount, dueDate }`.
- **Region label**: `$t("disputedInvoices.title")`.

### Estatement (`EstatementHtmlTable.vue`) — adjacent, not invoice-tab

Estatement is covered here because it uses `useInvoiceHtmlTableSelectionAndRows` with `tableVariant: "estatement"`, but it is **not** invoice-tab shaped:

- **Sort**: none on the HTML path (the legacy `mc-table` sort lives in the parent view).
- **Expansion**: none; there is no mobile expanded row and no presenter.
- **Columns are ordered and dynamic**: `getEstatementOrderedColumnIds(isSmallScreen, invoices)` returns the visible column IDs in display order; `<thead>` iterates a `visibleColumns` computed (`{ id, label, alignRight }[]`) and `<tbody>` uses `isColumnVisible(id)` guards so `<th>` and `<td>` stay aligned. Do **not** use a `Set` to drive the header order — `Set` iteration order is stable but not meant to be relied on for markup.
- **Data attributes**: Convention B (`data-column-id` / `data-cell-id`). Keep it.
- **Region label**: `$t("common.estatement")`.
- **Selection**: `useInvoiceSelection(pagedInvoices).handleSelectionChange`.
- **Styling**: the SFC defines its own header/body padding rules (`.estatement-table thead th` etc.) because the selection column shifts which cell is `:first-child`; do not regress to `:first-child` rules that assume an invoice-tab column order.

---

## Logic preservation — the list that every review checks

Every migration PR must confirm that these behaviors still fire from the new template/composables. Missing any is a regression, not an optimization.

1. **Sorting**: clicking a sortable `<th>` toggles ascending → descending → ascending, updates `aria-sort`, and writes to `invoicesStore.<tab>TabSortConfig`. On Disputed, the list refetches; on Open, virtualization rows update.
2. **Selection**: the header checkbox toggles all rows on the current page; row checkboxes toggle individually; `pageSelectionState` reflects partial selection with `indeterminate`; selection writes to the right store path (`invoicesStore.checkedInvoicesIds` directly for Disputed, through `useInvoiceSelection` elsewhere).
3. **Expansion**: on small screens the row expander opens the mobile expanded row; the presenter's `expandedData(row)` drives the list; no field the legacy template showed is lost.
4. **Desktop error row**: on Open/Paid/Credits, when `invoicesStore.expandedRowId === row.id`, the desktop notification row renders and `onCloseError()` clears it.
5. **Hover menu**: `row.id === hoveredId && <tab>Store.showHoverMenu` condition still gates the action buttons; `setActionableInvoice` / `clearActionableInvoice` update `invoicesStore.actionableInvoices`.
6. **Pagination**: page size select writes `appStore.resultsPerPage`; page change scrolls `.search-container` into view and tracks via `trackPaginationPageChange(page, route.name)`.
7. **Analytics**: every `copyButtonClicked(field, page)` call that existed in the legacy slot still fires from its `<mc-c-copy-item>` equivalent.
8. **Cancelled rows (Paid)**: cells carry `cell--cancelled`, the status cell stays interactive, and the mobile expanded row shows the "cancelled" prefix.
9. **PDF link + download**: `showInvoiceLink(row)` still gates the `show-link` class; clicking the invoice number still calls `invoicesStore.downloadInvoice(row)`.
10. **Division filter reset**: `resetDivisionFilter()` runs on the lifecycle hook the legacy table used (`onMounted` for Paid, `onUnmounted` for Open).

---

## DocumentReferenceCell prop contract

The legacy invoice tables pass every one of these props; all of them must survive migration unless a ticket says otherwise.

```vue
<DocumentReferenceCell
  :prefix-label="$t('tableHeadings.bl')"
  :BL="row.billOfLading"
  :HBL="row.documentReference"
  :customer-reference="getSecondaryCustomerRef(row) /* or row.customerReferenceNo ?? '' */"
  copy-field="billOfLading"
  copy-page="<open-invoice|paid-invoice|credits-invoice|disputed-invoice>"
  hbl-prefix-label=""
/>
```

Dropping `:HBL` silently breaks BL+HBL display and copy behavior; dropping `copy-page` breaks analytics.

---

## Anti-patterns (hard stops)

- Reimplementing selection, hover, expansion, or sort state in the SFC when the composable already provides it.
- Inlining a second copy of the sort button, row expander, or mobile expanded row when a sibling `*HtmlTable.vue` already ships it. Extend the shared layer.
- Inlining any `{ key, text }` expanded-row data in the SFC instead of extending `useInvoiceHtmlTablePresenters`. The one exception is cancelled-invoice prefix copy on Paid.
- Adding `cell--cancelled` or analogous state classes to a per-tab scoped stylesheet when they already live in the shared partial.
- Hard-coding colspans instead of computing them from `showSelectionColumn` + visible columns.
- Writing imperative shadow-DOM code (`tableRef.shadowRoot.querySelectorAll`). The HTML table has no shadow root; there is no parity required.
- Assuming Open's `expandedData` shape applies to Credits or Disputed.
- Mapping legacy `<mc-table>` props to `.mds-table--*` modifiers by guessing. Only `disablerowhighlightonhover` has an established, tested mapping in this repo (`mds-table--disable-row-highlight-on-hover` on the `section.mds-table`). Everything else goes through visual QA.
- Renaming `data-header-id` on a shipped invoice tab without updating `tests/unit`, `tests/component`, and `tests/e2e` in the same PR.
- Mixing `data-header-id` and `data-cell-id` in the same SFC.
- Wrapping native table markup with `:deep(...)` in the SFC's scoped styles.

---

## Tests

Follow **[test-structure](../test-structure/SKILL.md)**:

- Unit (Vitest): BDD `it` titles, feature-grouped `describe` blocks — Sorting, Selection, Pagination, Mobile expanded parity, Cancelled rows (Paid), API sort (Disputed), Virtualization (Open), Row hover/actions, Copy/analytics. No root-level `it`. Toggle `appStore.isSmallScreen` via Pinia when testing responsive branches.
- Component (Cypress): target headers and cells with the convention the SFC ships; use `:data-cy="row.id"` to scope per row. For virtualized tables, drive the test to scroll or stub the virtualizer.
- E2E: step definitions for invoice tabs currently target `data-header-id`. If you switch a table to `data-column-id` / `data-cell-id`, update every step in the same PR.

When a production bug is fixed:

1. The fix lands in the shared layer (composable, partial, presenter).
2. A failing-then-passing unit or component test is added.
3. This skill is updated only if the fix exposes a rule that was not already written down.

---

## After editing

Run targeted unit + component specs for the touched HtmlTable and its parent view, plus e2e if selectors changed. `npm run start` or `npm run build` catches missing `@use` paths after moves.
