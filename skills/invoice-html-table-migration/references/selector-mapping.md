# Selector mapping (DOM and e2e)

Selectors alone do not prove parity — always pair them with the behavior checks from **[../SKILL.md](../SKILL.md)**.

## Legacy `mc-table` slot hooks in tests — must change when the SFC migrates to `<table>`

**Problem.** Component and e2e tests often targeted cells with **slot** attributes on `<mc-table>`, for example `` `cy.get('[slot="12345_delete"]')` ``. After migration, cells are real `<th>` / `<td>` — those attributes **do not exist**. A migration PR that only updates the Vue SFC and leaves these selectors will fail in CI (timeouts looking for `[slot=…]`).

**What to do in the same PR as the SFC change:**

- Replace slot-based cell queries with the **[Step 3 — Emit the canonical HTML structure](../../html-table-components/SKILL.md#step-3--emit-the-canonical-html-structure)** contract: `tr` with `:data-cy="row.id"`, `td` with `data-header-id` (or `data-cell-id` per table convention), and any wrapper classes the SFC already uses (e.g. `.delete-column`).
- **Example (delete action on a migrated table):** scope with the table region class + row + action column, e.g. `'.pop-bl-table tr[data-cy="1"] .delete-column mc-button'` — not `` `[slot="1_delete"]` ``.
- **Sibling still on `mc-table`:** prefer region + row + class in the **light** DOM (e.g. `'.pop-invoice-table .delete-column mc-button'`) over slot name strings, so the next migration does not re-break the same lines.

Grep the repo for `` `[slot=` ``, `slot="`, and `` `*_delete` `` in `tests/` when you ship a table migration. See **html-table-components** Step 7 “Test selector migration” for the full checklist.

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
