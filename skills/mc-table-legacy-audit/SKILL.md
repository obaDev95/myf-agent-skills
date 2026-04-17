---
name: mc-table-legacy-audit
description: >-
  Pre-migration inventory of a legacy `<mc-table>` before any HTML rewrite. Produces the
  parity matrix (columns, slots, props, conditionals, behaviors, side-effects) that feeds
  the `html-table-components` migration pipeline. Use when starting any `<mc-table>` →
  `<table class="mds-table">` migration, auditing a shipped migration for missing parity,
  or when a user asks what the legacy table actually does.
---

# Legacy `<mc-table>` audit

Every HTML-table migration starts here. An audit that lists the wrong slot, drops a mobile-only field, or misses a shadow-DOM listener guarantees a downstream regression — no rewrite can recover information that was never captured. The audit's output **is** the parity matrix consumed by **[html-table-components/SKILL.md](../html-table-components/SKILL.md)** Step 2.

Run this skill **read-only**. No files are edited here.

---

## Scope of an audit

The legacy table may be:

- A checked-in `*Table.vue` SFC that still renders `<mc-table>`.
- An inline `<mc-table>` block inside a view (for example `src/views/EstatementTable.vue` before `EstatementHtmlTable.vue` landed).
- A file that no longer exists on `HEAD` because the migration already merged — load it at `git show <migration_commit>^:<path>`.

Anchors for past ui-myfinance migrations live in **[../html-table-components/references/ui-myfinance-tables.md](../html-table-components/references/ui-myfinance-tables.md)**.

---

## Loading a legacy baseline

```bash
# exact pre-migration revision when the migration is already merged
git show <migration_commit>^:src/components/FooTable.vue > /tmp/FooTable_legacy.vue

# follow renames if the path moved
git log --follow --oneline -- src/components/FooTable.vue

# first commit that introduced the HtmlTable file
git log --diff-filter=A --format=%H --reverse -- src/components/FooHtmlTable.vue | head -1

# earliest version of the feature that still used mc-table
git log --oneline -S 'mc-table' -- src/views/SomeView.vue
```

Always audit the **pre-migration** file, not the merged `HtmlTable.vue`. The migrated file is the hypothesis being checked; the legacy file is the ground truth.

---

## Output: the parity matrix

Produce one matrix per audited table. Do not skip sections even when "there is nothing" — "no selection column" is a legitimate row, silent omission is not.

### 1. `<mc-table>` attributes

List every attribute on the root `<mc-table>` element with its value. Common attributes to check for:

| Attribute | Meaning during migration |
|-----------|--------------------------|
| `sortmanual` | Sort is driven externally — expect `sortChange` handler writing to store |
| `sortdefaultcolumnid` / `sortdefaultdirection` | Initial sort column and direction — source store path |
| `select` / `selectrowdisabled` | Row selection enabled; rows matching predicate are unselectable |
| `expand` / `expandopened` | Expanded rows enabled; `expandopened` lists the currently expanded rows |
| `height` / `headersticky` | Scroll height and sticky header behaviour |
| `customstyles` | CSS string injected into the shadow DOM — must be re-expressed as scoped SCSS / row classes |
| `disablerowhighlightonhover` | Maps to `mds-table--disable-row-highlight-on-hover` on the HTML path |
| `data-*` / `class` / inline `style` | Carry over or re-express |

Every attribute that is not re-expressed on the HTML path is a dropped behaviour.

### 2. Columns (per breakpoint)

For every element of the `columns` array (and `mobileColumns` if the table swaps them by breakpoint), record every `TableColumn` field: `id`, `label`, `align`, `verticalAlign`, `width`, `rowspan`, `colspan`, `tabularFigures`, `sortDisabled`, `sortdisableoninitialload`, `noWrap`, `sorter`, `sticky`, `columns`, `footerColspan`, `renderAsHeader`, `dataType`, `description`, `subDataKey`, `subDataLabel`, `rowClickDisabled`, `headerTemplate`, `cellTemplate`, `truncate`.

Flag any field the migration must act on (for example `label: ""` requires an empty `<th>` with `sr-only`; `sortDisabled: true` means the sort button must not render).

### 3. Named slots

Every `:slot="\`${row.id}_<columnKey>\`"` and `:slot="\`${row.id}_expanded\`"` is a slot. For each:

- Record the child component(s) inside the slot.
- Record **every prop** passed to each child component — verbatim. Prop drift (for example `DocumentReferenceCell :HBL` omitted) is the single most common source of silent regressions in this codebase.
- Record the slot's own class and inline style attributes; re-express them as inner wrappers (never on the `<td>` itself).
- Record whether the slot is guarded by `v-if` / `v-else` — those conditionals become conditionals on the new `<td>` or on an inner branch.

### 4. Conditionals and breakpoints

Run these greps on the legacy SFC and record every match:

```bash
rg 'isSmallScreen|isTabScreen|smallScreenBreakPoint' <legacy>
rg 'v-if|v-else|v-show' <legacy>
rg 'customStyles|shadowRoot|manualSyncTableSelection' <legacy>
```

For every `v-if="appStore.isSmallScreen"` / `v-if="!appStore.isSmallScreen"` block, write down:

- Which surface it governs (column header, slot, row, inline fragment).
- What the user sees on that branch.
- Whether the inverse branch exists explicitly or is implied by omission.

**Mobile-only fields embedded inside a desktop column** (for example a status tag that renders inside an amount cell only on small screens) are the highest-risk audit target — they do not have their own `<th>` and are easy to miss when the column list is the only source of truth.

### 5. Behaviours and side-effects

| Behaviour | What to capture |
|-----------|-----------------|
| Sorting | `@sortchange` handler; store path it writes to; whether it triggers a list refetch |
| Selection | `@selectchange` handler; which store holds the selected ids; any `manualSyncTableSelection(tableRef)` calls after filter/page change |
| Hover menu | `@mouseenter` / `@mouseleave` on root; `tableRef.shadowRoot.querySelectorAll` listeners; gating conditions (`invoicesStore.showHoverMenu`, cancelled rows, etc.) |
| Expansion | `@expandchange` or store-driven `expandedRowId`; what content the expanded slot renders on mobile vs. desktop |
| Pagination | Which store owns `pages.number` / `totalPages`; analytics (`trackPaginationPageChange`); side-effects (`fetchDisputedStatus`, `getProofOfPaymentStatus`, etc.) |
| Cell-level events | `@click` on cell content; copy buttons (`mc-c-copy-item` `@itemcopied` analytics); `@click.stop` usage |
| Lifecycle | `onMounted` / `onUnmounted` / `onBeforeUnmount` — division filter resets, cancelled-invoice removals |
| Row classes | Per-row CSS state — hovered, selected, cancelled, expanded, disabled |

### 6. Presenter / helper functions

Legacy mobile details come from ad-hoc helpers (`expandedData`, `getSecondaryCustomerRef`, `formattedInvoicedAmount`, per-column `customStyles` builders). List them with their signature and what they return. The migration either moves these into the shared composable (`useInvoiceHtmlTablePresenters`) or into the SFC `<script setup>` — never duplicates them.

### 7. Tests that touch this table

Grep the test tree before migration so the DOM-hook convention is a conscious decision, not a surprise:

```bash
rg -l 'FooTable|data-header-id.*<columnKey>' tests/unit tests/component tests/e2e
```

Record every selector style currently in use (`data-header-id`, `data-cy`, `data-test`, slot-based queries) so the migration either preserves them or schedules the rename in the same PR.

---

## Sign-off

An audit is complete when **every** row of the parity matrix has a concrete answer (including "not present"). An audit that says "I could not find X" is not done — go look.

Deliver the matrix as a markdown artifact on the PR or in the migration ticket; `html-table-components/SKILL.md` Step 3 consumes it directly.

---

## Anti-patterns

- Auditing `HEAD`'s merged `HtmlTable.vue` instead of the legacy SFC. The migrated file is the hypothesis being tested; using it as ground truth is circular.
- Skipping mobile slots because the desktop column list does not mention them. `mc-table` frequently rendered mobile-only surfaces inside desktop columns — `isSmallScreen` is the only reliable signal.
- Trusting comments in the legacy file over the actual conditional expressions. Comments go stale; `v-if` branches do not.
- Merging the audit into Step 3 in one pass. Write the matrix first, then implement against it; otherwise the matrix is written to justify what was already coded.
- Auditing one tab and copy-pasting the matrix headings to a sibling tab without re-reading the sibling's legacy file. Open's matrix is not Credits' matrix.
