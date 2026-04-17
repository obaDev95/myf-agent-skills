---
name: migration-pr-review
description: >-
  Review an `<mc-table>` → HTML table migration PR against a fixed rubric: logic
  preservation, DOM / data-attributes, shared SCSS, tests, and docs. Use when reviewing
  an `*HtmlTable.vue` PR, auditing a merged migration for regressions, or triaging a
  visual / behavioural bug reported against a migrated table.
---

# Migration PR review

This skill is the reviewer-side complement to **[html-table-components/SKILL.md](../html-table-components/SKILL.md)** and **[invoice-html-table-migration/SKILL.md](../invoice-html-table-migration/SKILL.md)**. The migration skills tell an agent how to produce a migration; this skill tells a reviewer (agent or human) how to check one.

A review is a verification pass against the **legacy** file, not a read of the migrated file in isolation. Always diff against `git show <migration_commit>^:<legacy path>` (or the equivalent baseline produced by **[mc-table-legacy-audit](../mc-table-legacy-audit/SKILL.md)**) before asserting parity.

---

## Review workflow

1. **Load the legacy baseline.** Use the anchors in **[../html-table-components/references/ui-myfinance-tables.md](../html-table-components/references/ui-myfinance-tables.md)**, or run `git log --follow` to find the pre-migration SFC.
2. **Pick the variant.** Not every rubric bullet applies to every table — see **Variant checklist** below. Skipping a bullet marked N/A for the variant is not a gap.
3. **Run the migration-pipeline checks in order** (below). Do not skip sections — a passing check on one section does not imply the others pass.
4. **Classify findings** as block / follow-up / nitpick (see the bottom of this skill).
5. **Produce a summary** that cites the legacy line range for every asserted parity gap. `git blame` output or the `git show` hash + path is enough.

---

## Variant checklist

Which rubric bullets apply to which variant. Bullets marked **N/A** below may be skipped without a finding; bullets marked **yes** are required; bullets marked **conditional** fire only when the legacy template exposed the behaviour.

| Rubric bullet | Invoice tab (Open, Paid, Credits, Disputed) | Estatement | Refunds-selected |
|---------------|---------------------------------------------|------------|------------------|
| §1 Sorting | yes | N/A (no sort on table) | N/A unless config adds sort |
| §1 API sort refetch | Disputed only | N/A | N/A |
| §1 Header checkbox select-all + per-row | yes | yes | yes |
| §1 Cancelled-row select-all exclusion | Paid only | N/A | N/A unless config adds disabled rows |
| §1 Mobile expansion via `toggleMobileRow` + presenter | yes | N/A (no expansion) | N/A |
| §1 Desktop error row (`expandedRowId`) | Open, Paid, Credits (Disputed omits) | N/A | N/A |
| §1 Hover menu | yes | N/A (no row-level action menu) | conditional |
| §1 Pagination + `.search-container` scroll | yes (on the SFC) | N/A on the SFC (pagination lives in parent view `EstatementTable.vue`) | conditional |
| §1 Copy analytics | yes where `mc-c-copy-item` renders | yes | yes |
| §1 Lifecycle resets | conditional (tab-specific `resetDivisionFilter`, `REMOVE_CANCELLED_INVOICES`) | N/A | conditional |
| §2 Canonical wrapper + column order | yes | yes (no mobile expander column) | yes |
| §2 Paired `<th>` / `<td>` `v-if` | yes | yes (via `isColumnVisible` guard) | yes |
| §2 Data-attribute convention | yes (A) | yes (B) | yes (B) |
| §2 Sort / select / expander `data-test` | yes (sort + select always; expander when mobile) | `select-all-<tab>` only (no sort, no expander) | conditional |
| §2 Colspans breakpoint-aware | yes | N/A unless colspan rows are added | conditional |
| §2 `DocumentReferenceCell` prop contract | invoice tabs only | N/A (Estatement uses `MaskedText` / `mc-c-copy-item`) | conditional |
| §3 Every shared class has `@use` | yes | yes | yes |
| §3 No `:deep()` on native markup | yes | yes | yes |
| §3 No inline `style=""` for layout | yes (with carve-outs — see §3) | yes | yes |
| §3 No duplicated chrome blocks | yes | yes (Estatement-specific padding rules are legitimate, not duplicates) | yes |
| §4 Feature groups | full canonical list | Rendering + Selection on component spec; Pagination + Page size on parent-view spec (split coverage is acceptable for this variant) | applicable subset |
| §4 Variant-specific groups | per-variant (Cancelled rows / API sort / Virtualization) | Dynamic column order | N/A |
| §4 Selector rename hygiene | yes | yes | yes |
| §5 Inventory row in `ui-myfinance-tables.md` | yes | yes | yes |

When a table introduces a new pattern that this table does not cover, add a row to the matrix in the same PR. Do not silently skip bullets for a new variant.

---

## Section 1 — Logic preservation

Each of these must fire from the new template/composables and produce the same user-visible outcome as the legacy slot.

- [ ] Sorting toggles the right direction and writes to `invoicesStore.<tab>TabSortConfig`.
- [ ] When the tab refetches on sort change (Disputed today), both the store write **and** the refetch call land in the handler. Assert both.
- [ ] Header checkbox toggles all rows on the current page; row checkboxes toggle individually; `pageSelectionState.indeterminate` handles partial selection.
- [ ] For tables with cancelled-row or disabled-row rules (Paid today), `toggleSelectAll` must exclude those rows — CSS `pointer-events: none` only blocks direct clicks, not the programmatic select-all path. See **[../invoice-html-table-migration/SKILL.md](../invoice-html-table-migration/SKILL.md)** § Paid for the `rowSelectable` predicate contract.
- [ ] Selection writes to the correct store path — `invoicesStore.checkedInvoicesIds` directly for Disputed, through `useInvoiceSelection` elsewhere.
- [ ] Mobile expansion opens and closes via `toggleMobileRow` from `useInvoiceHtmlTableSelectionAndRows`. Content comes from `useInvoiceHtmlTablePresenters(variant).expandedData(row)`, not from a new hand-rolled list in the SFC.
- [ ] **`subDataKey` / `subDataLabel` / `dataType` subtext fields from the legacy column config are rendered.** These fields have no slot name in `mc-table` — a slot-centric review will miss them. For every legacy column with `subDataKey`, verify the HTML migration renders the field somewhere: inline inside the same `<td>`, inside the presenter's `expandedData(row)`, or both. See **[../mc-table-legacy-audit/SKILL.md](../mc-table-legacy-audit/SKILL.md)** § Columns per breakpoint for the audit contract; Credits `reason` is the canonical regression class.
- [ ] Desktop error row renders when `expandedRowId === row.id`, if the legacy template exposed a desktop error slot. If the legacy did not expose one (Disputed), the new template must not render one either.
- [ ] Hover menu gating still matches the legacy condition (`row.id === hoveredId && <tab>Store.showHoverMenu && !cancelled`). `setActionableInvoice` / `clearActionableInvoice` drive `invoicesStore.actionableInvoices`.
- [ ] Pagination size-change writes `appStore.resultsPerPage`, resets the current page, calls the right side-effect (`fetchDisputedStatus`, `getSortFilteredDisputedInvoiceList`, …), and scrolls `.search-container` into view.
- [ ] Analytics calls — every `copyButtonClicked(field, page)` and `trackPaginationPageChange(page, tab)` that existed in the legacy still fires from the new template.
- [ ] Lifecycle resets (`resetDivisionFilter`, `REMOVE_CANCELLED_INVOICES`, etc.) land in the same hook (`onMounted` vs `onUnmounted`) as the legacy.

A logic check that fails is always a **block**.

---

## Section 2 — DOM and data attributes

- [ ] Canonical wrapper: `<section class="<tab>-table mds-table mds-tabular-figures" :class="tableClasses" :aria-label="…">`.
- [ ] Column order: selection → mobile expander → data columns. No reshuffling unless the legacy reordered them.
- [ ] `<th>` and `<td>` `v-if` conditions are paired on identical expressions. Every mobile-only body cell has a matching mobile-only header.
- [ ] Data-attribute convention is consistent within this SFC: `data-header-id` on both `<th>` and `<td>` **or** `data-column-id` on `<th>` plus composite `data-cell-id` on `<td>`. Mixing is a block.
- [ ] Every row has `:key="row.id"` and `:data-cy="row.id"`.
- [ ] Sort buttons carry `data-test="sort-button-<columnKey>"`; the select-all checkbox carries the table's agreed `data-test`; the row expander carries `data-test="row-expander"`.
- [ ] Colspans are computed (`desktopColspan`, `mobileExpandedColspan`) — never hard-coded.
- [ ] **Colspan arithmetic matches `<th>` count at the breakpoint where the colspan-bearing `<tr>` renders.** Manually count the `<th>` that render at that breakpoint (desktop: selection + all data columns with non-falsy `v-if`; mobile: selection (if shown) + expander + mobile-visible data columns). Common shipped bugs: `desktopColspan` used on virtual-padding rows that also render on mobile (over-spans on small screens), or `data` column count miscounted on desktop (under-spans by one). See **[../html-table-components/SKILL.md](../html-table-components/SKILL.md)** § Step 3 "Colspans are computed per breakpoint".
- [ ] No `<th>` / `<td>` uses `display: flex` or `display: grid` directly. Flex/grid stacks live on inner wrappers.
- [ ] Every `DocumentReferenceCell` / `SubCell` / copy-item preserves every legacy prop (`BL`, `HBL`, `customer-reference`, `copy-field`, `copy-page`, prefix labels). Dropped props are a block.
- [ ] No imperative `tableRef.shadowRoot.*` code. The HTML table has no shadow root.

---

## Section 3 — Shared SCSS

- [ ] Every shared class used in the template has a corresponding `@use` line in `<style scoped>`. A template that renders `column_right` without `@use "scoped-styles/invoice-table"` ships unstyled.
- [ ] **Leaf cell SFCs render inside `<td>` carry their own `@use`.** Vue scoped styles are per-SFC; the parent `*HtmlTable.vue`'s `@use` does not reach child DOM. Audit `OpenInvoiceAmountCell.vue`, `OpenInvoiceDueDateCell.vue`, `SubCell.vue`, `DocumentReferenceCell.vue`, and any new `*Cell.vue` — if their templates reference `column_right`, `align-right`, `adjust-width`, or any class defined in `invoice-table.scss`, the leaf's own `<style scoped lang="scss">` must `@use` the partial. See **[../shared-table-scss-refactor/SKILL.md](../shared-table-scss-refactor/SKILL.md)** § "Leaf cell components have their own `@use`".
- [ ] No `:deep()` on native table markup in this SFC. Same-SFC scoped styles apply without it.
- [ ] No inline `style=""` attributes for layout — only the dynamic `tableInlineStyles` computed for an explicit `height` prop. **Permitted exceptions**: (a) the canonical currency-spacing pattern `style="padding-left: 4px"` on the currency code span inside a stacked amount cell, shown in **[../html-table-components/SKILL.md](../html-table-components/SKILL.md)** § Stacked numeric cells — acceptable until replaced by a utility class; (b) virtual-row spacers used by `useWindowVirtualTable`, which need dynamic `height`, `padding: 0`, and `border: none` to match the virtualizer output; (c) any other dynamically computed dimension bound to a prop or reactive value. All three exceptions carry **dynamic** values that cannot be expressed as a static class. Static decorative inline styles are still a block.
- [ ] `<style scoped>` does not duplicate a block (row-expander, sort-button, expanded-cell, desktop-notification-row, row-state) that a sibling `*HtmlTable.vue` already ships. If duplication appears, the PR either extracts to the shared partial in the same commit or the reviewer opens a follow-up issue with a clear owner. See **[../shared-table-scss-refactor/SKILL.md](../shared-table-scss-refactor/SKILL.md)**.
- [ ] The `<section>` wrapper's scoped CSS is the only thing that truly is SFC-specific. Anything else is either in the shared partial or a per-tab partial (`credits-table.scss`).

---

## Section 4 — Tests

Follow **[test-structure](../test-structure/SKILL.md)** and its **[html-table-feature-groups](../test-structure/references/html-table-feature-groups.md)** reference.

- [ ] No root-level `it(...)` in the component, view, or Cypress spec — every test sits under a feature-named `describe`.
- [ ] Feature groups match the canonical list: Sorting, Selection, Pagination, Page size, Rendering, Mobile expanded parity, plus the variant-specific groups (Cancelled rows for Paid, API sort for Disputed, Virtualization for Open).
- [ ] Responsive branches are covered by toggling `appStore.isSmallScreen` in Vitest and by at least one Cypress run at a small viewport when the mobile template diverges from desktop.
- [ ] Virtualized tables (Open) do not assert "row count in DOM equals page size". Tests scroll the container, stub the virtualizer, or assert against the page store.
- [ ] Cancelled-row tests (Paid) assert both the `cell--cancelled` class and the absence of an interactive handler (no selection change, no download).
- [ ] API-sort tests (Disputed) assert **all three**: the `invoicesStore.changeSort("disputedTab", …)` write, the `disputesStore.UPDATE_SORT_VALUE({ sort_by, id })` write with the UI-column-to-API-field mapping applied, and the awaited `disputesStore.getSortFilteredDisputedInvoiceList()` refetch. Shipped Vitest specs that only assert `UPDATE_SORT_VALUE` + refetch miss the `changeSort` half of the dual-write and will not catch regressions where `disputedTabSortConfig` falls out of sync with the API fetch.
- [ ] When the PR renames `data-header-id` → `data-cell-id` on a shipped table, every unit / component / e2e selector is updated in the same PR. Partial renames are a block.

---

## Section 5 — Docs and follow-ups

- [ ] Intentional UX / API changes are documented in the PR description, not discovered by the next reviewer.
- [ ] New or modified shared utilities (partial classes, composable exports, presenter fields) are consumed on every relevant table in the same PR, or a follow-up ticket is opened with a named owner.
- [ ] References under `html-table-components/references/ui-myfinance-tables.md` are updated when a new table ships or a migration commit anchor changes.

---

## Classifying a finding

| Class | Trigger | Action |
|-------|---------|--------|
| **Block** | Logic preservation gap; dropped prop; mixed data-attribute conventions; partial selector rename; missing `@use` for a consumed shared class; inline shadow-DOM code. | Do not merge until resolved. |
| **Follow-up** | Duplicated scoped CSS that was not extracted in this PR; stale comment; sub-optimal test grouping that still passes. | Accept the PR; create a ticket with a named owner before closing the review. |
| **Nitpick** | Whitespace, comment tone, preference-level naming. | Optional comment; do not block. |

A reviewer who cannot distinguish the three is slower than no reviewer.

---

## Output template

Conclude the review with the following structure so the author can triage fast:

```
## Blocks
- <section> — <what, with legacy line range or git show path>

## Follow-ups
- <section> — <what, with suggested ticket title>

## Verified (no action)
- <section list>

## Notes
- <intentional UX / API changes acknowledged>
```

No verdict is the worst verdict — always produce one of these four sections.
