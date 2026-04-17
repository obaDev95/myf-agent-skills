# HTML-table feature groups

Canonical `describe` blocks for every migrated `*HtmlTable.vue` spec (Vitest unit + component, Cypress component). Pair with the main **[../SKILL.md](../SKILL.md)** rules (BDD titles, no root-level `it`, no `describe("snapshots")`).

The goal is that failures map to **functionality**, not to file mechanics, and that two reviewers reading two different `*HtmlTable.spec.ts` files see the same shape.

---

## Mandatory groups (every migrated HTML table)

| `describe` name | Typical assertions |
|------------------|--------------------|
| `Rendering` | Headers exist; sortable headers render a sort button, non-sortable render a plain label; selection / expander columns appear when expected; initial row count matches page data. |
| `Sorting` | Clicking a sortable header toggles direction; `aria-sort` updates; the store `changeSort("<tab>Tab", column, direction)` is called with the expected args; re-clicking the same column cycles; clicking a non-sortable header is a no-op. |
| `Selection` | Header checkbox toggles all rows on the current page; row checkboxes toggle individually; `pageSelectionState.indeterminate` surfaces when a subset is selected; the right store path is written. |
| `Pagination` | Page change updates the current page; scrolls `.search-container`; invokes `trackPaginationPageChange(page, routeName)`; triggers the tab's side-effect (`fetchDisputedStatus`, `getSortFilteredDisputedInvoiceList`, …). |
| `Page size` | Size change writes `appStore.resultsPerPage`, resets current page, and invokes the tab's size-change side-effect. |
| `Mobile expanded parity` | With `appStore.isSmallScreen = true`, the row expander is visible, toggling opens the `mds-table__expanded-row`, `expandedData(row)` entries render as `data-dl` pairs, and every field the legacy mobile slot exposed appears somewhere in the new template (presenter or inline). |

These six groups appear on **every** invoice-domain HtmlTable spec. A spec missing any one of them is incomplete.

---

## Variant-specific groups

Add these only when the variant warrants it; the migration skill calls out which apply.

| Group | Applies to | Assertions |
|-------|-----------|------------|
| `Cancelled rows` | Paid | `isCancelledPaymentInfo(row)` cells render the `cell--cancelled` class; selection is disabled; download link is suppressed; mobile expanded row shows the cancelled-invoice prefix; status-cell tooltip renders. |
| `API sort` | Disputed | Column click writes both `invoicesStore.changeSort("disputedTab", …)` **and** `disputesStore.UPDATE_SORT_VALUE({ sort_by, id })`, then awaits `getSortFilteredDisputedInvoiceList()`. |
| `Virtualization` | Open | With a scroll container and a large `pagedInvoices` set, only a window of rows is mounted; `paddingTop` / `paddingBottom` virtual rows exist; scrolling advances the window. Use test doubles for `useWindowVirtualTable` when the real virtualizer would flake. |
| `Desktop error row` | Open, Paid, Credits | When `invoicesStore.expandedRowId === row.id` on desktop, the `desktop-notification-row` renders; clicking close fires `onCloseError()` which clears `invoicesStore.expandedRowId`. |
| `Hover menu` | All invoice tabs | `@mouseenter` sets `hoveredId`; when `<tab>Store.showHoverMenu` is true and the row is not cancelled (Paid), `ActionButtons` renders; leaving clears it. |
| `Copy analytics` | All invoice tabs with `mc-c-copy-item` | Triggering `@itemcopied` on each copy item fires `copyButtonClicked(field, page)` with the page tag of the tab. |
| `Dynamic column order` | Estatement | `getEstatementOrderedColumnIds(isSmallScreen, invoices)` drives which `<th>` / `<td>` render; `<thead>` and `<tbody>` stay aligned when the set of visible columns changes. |

Groups that do not apply to the table under test are omitted — do not add empty `describe` blocks.

---

## Skeleton

```ts
describe("<TableName>HtmlTable", () => {
  describe("Rendering", () => {
    it("when mounted at desktop width, then renders sortable headers with sort buttons and non-sortable as labels", () => { /* … */ });
    it("when the selection column is hidden, then the row-selector `<th>` and `<td>` are not rendered", () => { /* … */ });
  });

  describe("Sorting", () => {
    it("when a sortable header is clicked, then aria-sort reflects the new direction", () => { /* … */ });
    it("when the active column is clicked again, then the direction toggles", () => { /* … */ });
    it("when a non-sortable header is clicked, then no sort action fires", () => { /* … */ });
  });

  describe("Selection", () => { /* … */ });
  describe("Pagination", () => { /* … */ });
  describe("Page size", () => { /* … */ });

  describe("Mobile expanded parity", () => {
    beforeEach(() => setSmallScreen(true));
    it("when a row is expanded, then every presenter field renders as a data-dl pair", () => { /* … */ });
    it("when the legacy template rendered an inline mobile field, then it still renders", () => { /* … */ });
  });

  // Variant-specific groups below (Cancelled rows, API sort, Virtualization, …)
});
```

---

## Asserting `<script setup>` behaviour

`*HtmlTable.vue` files use `<script setup lang="ts">`. Unlike Options API components, **`<script setup>` does not auto-expose its bindings on `wrapper.vm`**. Only names passed to `defineExpose({ ... })` are reachable from tests via the component instance.

Prefer these assertion paths, in order:

1. **Drive behaviour through the DOM and assert the outcome.**
   ```ts
   await wrapper.find('[data-test="sort-button-invoiceNo"]').trigger("click");
   expect(invoicesStore.changeSort).toHaveBeenCalledWith("paidTab", "invoiceNo", "ascending");
   ```
   This is the default — it exercises the real event path and survives refactors of the setup block.

2. **Assert on the store / composable spies** for side-effects the SFC handler fires. Mock the store with `createTestingPinia({ stubActions: false })` (or the project helper) and spy on the action.

3. **Only `defineExpose` a handler when tests genuinely need to invoke it out-of-band** (Paid exposes `setActionableInvoice` / `clearActionableInvoice` because the parent view needs them, and the tests ride on that same expose). Exposing *only* for tests is a smell — document the reason in the SFC with a comment.

Shipped specs that call `wrapper.vm.sortChange(...)`, `wrapper.vm.paginationChange(...)`, `wrapper.vm.toggleSelectAll()` on a `<script setup>` SFC without a corresponding `defineExpose` are relying on Vue test-utils internals that are not contract — the calls may break on a minor test-utils upgrade. When you touch such a spec, rewrite the assertion to go through the DOM + store path.

---

## Anti-patterns

- Collapsing Sorting and Selection into one `describe("Interactions")`. Reviewers cannot tell which feature broke when a failure shows up.
- Using `describe("snapshots")` with a single `it` that deep-renders the component. The test name says nothing about behaviour; snapshot churn goes unreviewed.
- A root-level `it(...)` directly under the `describe("<TableName>HtmlTable")` block. Every assertion belongs to a feature.
- Skipping `Mobile expanded parity` because the test file was originally desktop-only. Every migrated table now has a mobile branch — assert it.
- Copy-pasting a sibling table's `Variant-specific` groups without re-reading the variant. Open's `Virtualization` does not apply to Paid; Paid's `Cancelled rows` does not apply to Credits.
