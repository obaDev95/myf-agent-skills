# Selector mapping (E2E / DOM)

Use with the main skill’s parity sections—**selectors alone do not prove behavioral parity**.

## Structural references in repo

**`src/components/OpenInvoicesHtmlTable.vue`** is the primary **behavioral** reference (sorting, selection, virtualization). It and the other shipped invoice `*HtmlTable.vue` files still use **`data-header-id`** on `<th>` and `<td>` for most tests and e2e—align step definitions with that unless you are doing a ticketed rename.

**Virtualization:** Open’s table only renders a **window** of rows. Assertions that count `<tr>` nodes or query “all invoice rows” may fail unless the test scrolls the scroll container or stubs `useWindowVirtualTable`—see the main skill’s **Window virtualization (Open invoices)** section.

**`src/components/RefundsSelectedHtmlTable.vue`** demonstrates the **newer** hook pattern: **`data-column-id`** on headers and composite **`data-cell-id`** on body cells—see **`html-table-components`**. Use it when adding **new** selectors or refreshing a table intentionally, not as proof that Open/Paid/Credits/Disputed already use those attributes.

## Expanded / mobile content

**Expanded row parity** is not only “can Cypress find a cell?” Legacy `mc-table` often showed fields **only** in mobile or expanded slots. After migration, the same data must still surface via `expandedData(row)` (and related helpers) from `useInvoiceHtmlTablePresenters` in `InvoiceHtmlTablePresenters.composable.ts`. Also verify **inline** mobile cells where the product put copy outside the presenter list. If a field existed in the legacy mobile template but never appears in the new template or presenters, users lose information even when row data is present—verify against the **legacy template**, not only selectors.
