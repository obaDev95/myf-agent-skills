# Selector mapping (DOM and e2e)

Selectors alone do not prove parity — always pair them with the behavior checks from **[../SKILL.md](../SKILL.md)**.

## Conventions in the shipped tables

| SFC | Header / cell convention | Notes |
|-----|--------------------------|-------|
| `OpenInvoicesHtmlTable.vue`, `PaidInvoicesHtmlTable.vue`, `DisputedInvoicesHtmlTable.vue`, `credits/CreditsHtmlTable.vue` | `data-header-id="<columnKey>"` on both `<th>` and `<td>` | Aligns with invoice unit/component/e2e specs. Do not flip to `data-column-id` / `data-cell-id` without a PR that also updates every test and step definition that greps for `data-header-id`. |
| `EstatementHtmlTable.vue`, `RefundsSelectedHtmlTable.vue` | `data-column-id="<columnKey>"` on `<th>` + `:data-cell-id="\`${row.id}-<columnKey>\`"` on `<td>` | Demonstrates the refreshed pattern — use for new tables or an intentional refresh, not as proof that the invoice tabs already moved. |

## Always present (regardless of convention)

- `<tr :data-cy="row.id">` on every body row. Scopes per-row queries.
- `<tr :data-cy="`expanded-${row.id}`">` on both the mobile expanded row and the desktop notification row.
- `<section data-test="table">` (or a tab-specific equivalent such as `data-test="estatement-table"`) on the wrapper.
- `<button data-test="sort-button-<columnKey>">` on every sort trigger.
- `<mc-checkbox data-test="select-all-invoices">` / `"select-all-estatement"` etc. on the header checkbox.
- `<button data-test="row-expander">` on the mobile expander button.

## Virtualization (Open)

`OpenInvoicesHtmlTable.vue` uses `useWindowVirtualTable`; only a window of rows renders at a time plus two virtual padding `<tr data-test="virtual-padding-top|bottom">`. Assertions that count `<tr>` nodes or look for "every row" on the page must either scroll the scroll container, widen the Cypress viewport, or stub the virtualizer.

## Mobile expanded parity

Fields rendered only in the legacy mobile/expanded slots must surface either through `useInvoiceHtmlTablePresenters(variant).expandedData(row)` (preferred) or as an inline mobile block inside an existing column (only when the legacy template did this). When adding a new field, update the presenter first; extend the SFC inline only when the legacy UX was inline.
