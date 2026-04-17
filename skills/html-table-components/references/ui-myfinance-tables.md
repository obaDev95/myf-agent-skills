# ui-myfinance: migrated HtmlTable inventory

Repository-specific file names and migration commit anchors. Loaded alongside the generic **[../SKILL.md](../SKILL.md)** when you need an exact filename or legacy baseline; do not copy these paths into portable guidance for other projects.

## Loading a legacy baseline

```bash
git show <migration_commit>^:<legacy path>
```

The caret (`^`) resolves to the parent of the migration commit — that is the last revision of the legacy file before deletion. If the path does not exist at that commit, chase renames with `git log --follow -- <path>`. When a feature was introduced as a new `*HtmlTable.vue` with no same-path legacy SFC (e.g. `RefundsSelectedHtmlTable.vue`), diff against the parent view or an earlier branch that still carried `<mc-table>`.

## Migrated tables

| Current SFC | Legacy SFC (at `<commit>^`) | Migration commit | DOM-attribute convention | Variant quirks (covered in skills) |
|-------------|-----------------------------|-------------------|--------------------------|-------------------------------------|
| `src/components/OpenInvoicesHtmlTable.vue` | `src/components/OpenInvoicesTable.vue` | `609ced126` | `data-header-id` | Window virtualization; `useOpenInvoicesTableContext`; `isColumnSortable` gate; `scrollContainerRef`. |
| `src/components/PaidInvoicesHtmlTable.vue` | `src/components/PaidInvoicesTable.vue` | `d68735dd2` | `data-header-id` | `isCancelledPaymentInfo` cell class + tooltip + mobile prefix; `defineExpose({ setActionableInvoice, clearActionableInvoice })`. |
| `src/components/credits/CreditsHtmlTable.vue` | `src/components/credits/CreditsTable.vue` | `93cf9a553` | `data-header-id` | Precomputed `formattedCredit` / `formattedDate`; inline `row.reason` on mobile + desktop; nested `@use "./credits-table"` plus `@use "../scoped-styles/invoice-table"`. |
| `src/components/DisputedInvoicesHtmlTable.vue` | `src/components/DisputedInvoicesTable.vue` | `a512f38b1` | `data-header-id` | Dual-write sort (`invoicesStore.changeSort` + `disputesStore.UPDATE_SORT_VALUE` + list refetch); no desktop error row. |
| `src/components/EstatementHtmlTable.vue` | `src/views/EstatementTable.vue` (inline `<mc-table>`) | `12604f1a8` | `data-column-id` / `data-cell-id` | Ordered `visibleColumns` array driven by `getEstatementOrderedColumnIds`; no sort, no expansion; selection column shifts `:first-child` — SFC-local padding rules. |
| `src/components/RefundsSelectedHtmlTable.vue` | Introduced as HTML (no same-path legacy) | `674ef7be0` (intro); follow-ups `739433294`, `237ee6f6b` | `data-column-id` / `data-cell-id` | Empty actions `<th>` for legacy `label: ""`; `mds-table--disable-row-highlight-on-hover` because legacy config had the prop. |

## Shared foundations in this repo

- **Composables** — `src/composables/InvoiceHtmlTable.composable.ts` (`useInvoiceHtmlTableSelectionAndRows`, `useInvoiceHtmlTableSortHelpers`, `bindPageActionableInvoices`), `src/composables/InvoiceHtmlTablePresenters.composable.ts` (`useInvoiceHtmlTablePresenters`), `src/composables/InvoiceSelection.composable.ts` (`useInvoiceSelection`), `src/composables/OpenInvoicesTable.composable.ts` (`useOpenInvoicesTableContext`), `src/composables/WindowVirtualTable.composable.ts`.
- **Shared SCSS partial** — `src/components/scoped-styles/invoice-table.scss` carries `column_right`, `data-dl`, `mobile-actions`, `show-link`, `line-height-20`, `align-right`, `adjust-width`. Extend this file when you catch another table re-declaring the same rule.
- **Credits-specific partial** — `src/components/credits/credits-table.scss` carries credits-only data-dl widths and action-wrapper styles.

When you add a new migrated table, append a row above with the migration commit so the next agent can `git show <commit>^:…` the baseline.
