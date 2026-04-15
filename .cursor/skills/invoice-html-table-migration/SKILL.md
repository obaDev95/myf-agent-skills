---
name: invoice-html-table-migration
description: >-
  Step-by-step migration of invoice-related views from Maersk mc-table to semantic HTML
  `<table>` with `.mds-table` in ui-myfinance. Covers composables (InvoiceHtmlTable),
  mobile/expanded parity, conditional render audits, BDD feature-grouped tests, column/slot
  mapping, and e2e selectors. Use when migrating Open/Paid/Credits/Disputed tables,
  replacing mc-table, or when the user mentions invoice HTML table migration.
---

# Invoice tables: `mc-table` → HTML `<table>` (ui-myfinance)

## Version control

Invoice migration skills and their reference docs live under **`.cursor/skills/`** (for example **`invoice-html-table-migration/`** with **`references/`**). Commit those paths so the skill survives branch switches, rebases, and fresh clones. Treat the repo copy as the source of truth for the workflow; extend or fix the skill by editing tracked files and opening a PR like any other documentation.

## Recommended skill stack (best migration output)

Cross-checks against shipped `*HtmlTable.vue` files and parallel skill reviews agree on the following.

- **Primary (orchestrator):** **`invoice-html-table-migration` (this file)** — composables, tab-specific wiring, mobile/expanded parity, tests, and when DOM hooks may change. It is **not** sufficient alone: MDS presentation and app selector rules live in the two skills below.
- **Always pair with:** **`mds-component-table-html`** — `.mds-table` modifiers, sticky/scroll, numeric cells, selection/expansion UX, a11y patterns. Use it as a **modifier and interaction catalogue**, not as the only source of **wrapper shape** (see below).
- **Selector strategy:** **`html-table-components`** — `data-column-id` / `data-cell-id` for **new** or **ticketed** refreshes; **`data-header-id`** for incremental migrations that must match existing invoice tests/e2e until grep updates land.

**Workflow ranking (for correct invoice parity):** **Invoice-led (this skill + the two above)** beats **MDS-only** — `mds-component-table-html` alone misses `useInvoiceHtmlTable*`, presenters, tab stores, and repo test conventions. A **late pass** that blindly applies `data-column-id` / `data-cell-id` from `html-table-components` without a ticketed rename risks **selector churn** against shipped invoice tables; follow this file’s DOM-hook rules first.

**Ground-truth shell in this app:** shipped invoice tables (e.g. **`OpenInvoicesHtmlTable.vue`**) typically use **`<section class="… mds-table …">`** with **`aria-label`** / **`data-test`**. The MDS skill’s generic **`div.mds-table` + `<caption>`** example is a reference for modifiers and a11y ideas—match **existing invoice/refund `*HtmlTable.vue`** wrappers when in doubt.

## Skills to load first

1. **`mds-component-table-html`** — MDS `.mds-table` modifiers, sticky/scroll, selection/expansion, a11y checklist, Vue scaffolding.
2. **`html-table-components`** — App conventions: `data-column-id` / `data-cell-id`, tests; when **not** to bulk-rename legacy `data-header-id`.

## What you are replacing

- **Before:** `<mc-table>` (or legacy table component) driven by **column config** (`id`, labels) and **named slots** per row/column (e.g. `` `${row.id}_invoiceNo` ``).
- **After:** Plain `<table>`, `<thead>`, `<tbody>`, explicit `<tr>` / `<td>` per row, with **`section`** (or documented wrapper) carrying `aria-label` / `data-test` as needed.

Map **every** column and **every** conditional (mobile-only columns, expanders, row actions) before coding.

### Quick map: mc-table features to HTML

| Legacy `mc-table` | Typical HTML replacement in this repo |
|-------------------|----------------------------------------|
| `sortmanual` + `@sortchange` | `useInvoiceHtmlTableSortHelpers` + sort buttons on `<th>`, `aria-sort` |
| `select` + `@selectchange` | `<mc-checkbox>` in header/first column + `useInvoiceHtmlTableSelectionAndRows` |
| `expand` + `expandopened` + expanded slot | Extra `<tr>` rows (`mds-table__expanded-row`, `desktop-notification-row`, …) driven by expanded state |
| `customStyles` (incl. per-row CSS via `data-cy` on `tr`) | Scoped SCSS + row/cell classes (e.g. cancelled rows) or `:deep` on `.mds-table` shell—**re-express** the same UX, not necessarily the same string injection |
| Shadow-root row/header hover listeners | `@mouseenter` / `@mouseleave` on `<tr>` / composable helpers |
| `height` / sticky header props | `section` `style` / classes `mds-table--scrollable`, `mds-table--header-sticky` |

### `mc-table`-only props vs plain HTML table + MDS (no blind parity rule)

Several `<mc-table>` attributes (for example **`disablerowhighlightonhover`**) are implemented **inside that web component**—they adjust shadow DOM or internal styles, not something this repo can spell out line-for-line.

After migration, row chrome and hover behavior come from **`@maersk-global/mds-foundations`** (and related bundles) on `.mds-table`. **Do not** treat a 1:1 **prop → `.mds-table*` modifier** mapping as a skill or review requirement: we have **no** stable, skill-level guarantee of “equivalent” visuals across mc-table internals and a given foundations version. If product cares about a specific hover or row treatment, resolve it with **visual QA / design**, not by assuming a modifier name from the Maersk article.

**This skill does not reinforce** “must match legacy mc-table hover/disable props on the HTML table path” as a default checklist item.

## Critical behavior (read first)

- Preserve **small-viewport** behavior: anything the legacy table showed only in **mobile** or **expanded** rows must still appear after migration—do not assume another tab’s `expandedData` shape applies to credits/disputed/etc.
- **Mobile conditionals** are a **logic migration**, not only markup: pair `<th>` / `<td>` `v-if`s where the legacy table did; match **`appStore.isSmallScreen`** (and `!isSmallScreen`) to the legacy breakpoints, and `rg`/diff the legacy template before trusting comments. Some mobile-only fields intentionally sit **inside** an existing column (e.g. status or extra copy in an amount cell) **without** a dedicated header—validate user-visible outcomes, not only a strict one-header-per-cell grid.

## Shared code (invoice flows)

Wire behavior through existing composables instead of reimplementing:

| Concern | Where |
|--------|--------|
| Row selection, hover, mobile expand, `actionableInvoice`, row classes | `useInvoiceHtmlTableSelectionAndRows` in `src/composables/InvoiceHtmlTable.composable.ts` (pass correct `tableVariant`: `open` \| `paid` \| `credits` \| `disputed`) |
| Sort UI helpers (`aria-sort`, sort buttons) | `useInvoiceHtmlTableSortHelpers` from the same file |
| Expanded row / mobile detail content | `useInvoiceHtmlTablePresenters` in `src/composables/InvoiceHtmlTablePresenters.composable.ts` |
| Tab-specific data (pagination, store, filters) | **Open** often uses `useOpenInvoicesTableContext` and related composables. **Paid / Credits / Disputed** frequently wire stores, pagination, and handlers **in the SFC**—shapes differ; read the tab you migrate. **Paid** and **Credits** may use **`useInvoiceSelection`** with row selection; **Disputed** uses its own **sort** pipeline (UI column → API sort + list refetch). |

**Reference implementations** (patterns, not necessarily the newest data-attribute convention):

- `src/components/OpenInvoicesHtmlTable.vue`
- `src/components/PaidInvoicesHtmlTable.vue`
- `src/components/credits/CreditsHtmlTable.vue`
- `src/components/DisputedInvoicesHtmlTable.vue`

**DOM hooks in the repo today:** the four invoice `*HtmlTable.vue` files above still use **`data-header-id`** on `<th>` and `<td>` (legacy convention aligned with most invoice tests and e2e). Use **`data-column-id` / `data-cell-id`** only for **new** tables or an agreed, ticketed refresh—see **`html-table-components`**—and plan grep updates across `tests/` if you rename.

**Reference roles:** **Open + siblings** — behavioral parity, current invoice **`data-header-id`** selectors, and tab quirks ([Open vs other tabs](#open-vs-other-tabs-wiring-differs)). **`RefundsSelectedHtmlTable.vue`** — may use a **subset** of composables; demonstrates **`data-column-id` / `data-cell-id`** (per **`html-table-components`**) when you intentionally refresh DOM hooks beyond the legacy invoice pattern.

## Open vs other tabs (wiring differs)

Do not assume one invoice HtmlTable is a drop-in of another.

- **Open** may use **`useOpenInvoicesTableContext`**, **window virtualization** (`useWindowVirtualTable`, visible indices into `pagedInvoices`), and **`isColumnSortable`** from sort helpers to gate sort UI. Template iteration can be **index-based**, not only `v-for="row in pagedInvoices"`.
- **Paid** may **`defineExpose`** actionable-invoice helpers for parents; **Paid** mobile expanded rows can include **non-presenter** blocks (e.g. cancelled payment copy) before presenter-driven `expandedData`.
- **Paid / Credits** often pair selection with **`useInvoiceSelection`** patterns; **Disputed** selection and sort wiring can omit pieces other tabs use—match the target file.
- **Disputed** does **not** necessarily mirror the desktop **`expandedRowId` / `desktop-notification-row`** error pattern used on Open, Paid, and Credits—verify against the legacy disputed table, not Open alone.
- **Pagination / page-size constants** can differ by tab (`resultsPerPage` vs `resultsPerPageNew`, etc.)—copy constants from the tab you touch.
- **Wrapper test hooks:** several tabs use `data-test="table"` on the `<section>`; Open’s outer structure can differ (refs, height, virtualization)—update selectors per component.

### Slot-to-cell prop audit (parity)

Legacy `mc-table` slots are **named per row and column** (`` `${row.id}_${columnId}` ``). After migration, each slot’s inner markup becomes explicit `<td>` content. **Carry every prop** from the legacy slot into the same leaf components unless you have a tracked refactor.

- **`DocumentReferenceCell`:** match **`:BL`**, **`:HBL`** / `documentReference`, **`:customer-reference`**, **`copy-page`**, and prefix props to the legacy slot—dropping **`HBL`** when the old table passed it breaks BL+HBL display and copy behavior.
- **Cells that read context internally** (e.g. `OpenInvoiceStatusCell` calling `useOpenInvoicesTableContext()` for `actionableInvoice`) may not need duplicate props from the parent—confirm against the **child** implementation, not only the old template.
- **`mc-table` + shadow DOM:** the old code often attached **imperative** listeners (`tableRef.shadowRoot`, `manualSyncTableSelection`). The HTML table path uses **declarative** `tr`/`thead` handlers and composables—re-check selection sync after **filter** or **data** changes (division switch, page size, etc.).

### Window virtualization (Open invoices)

`OpenInvoicesHtmlTable.vue` uses **`useWindowVirtualTable`** so **only a subset** of page rows exists in the DOM at once. Component/e2e tests that assumed **every** `pagedInvoices` row mounts simultaneously may need **scroll**, **larger viewport**, or **test doubles** for the virtualizer—do not treat “row count in DOM === page size” as a given.

## Mobile expanded row parity (production lesson)

Credits (and similar) mobile details come from `expandedData(row)` → `useInvoiceHtmlTablePresenters` (e.g. `expandedDataCredits` in `InvoiceHtmlTablePresenters.composable.ts`). A production issue occurred when that list omitted fields the legacy `mc-table` showed only in mobile/expanded UI—e.g. **reason** (`CreditInvoice.reason`) and **type** copy—so data existed on the row but disappeared for users. **Credits caveat:** some fields (e.g. **reason**) may also render in **inline** small-screen cells outside `expandedData`—audit **presenters and** template `v-if` branches. **Disputed** may rely on mobile `expandedData` without the desktop **`expandedRowId` / notification** second row that other tabs use—do not assume the same row structure as Open/Paid/Credits.

**Verification (before merge):**

1. Diff the legacy template’s mobile/expanded slots against the new presenter output **and** inline mobile markup for this **domain** (not Open/Paid blindly).
2. Use `git show` / history on the pre-migration component if needed.
3. Align labels with `tableHeadings.*` / i18n keys users saw before.
4. Add Cypress (small viewport) and/or Vitest coverage for **expand or mobile block** when behavior differs from desktop.

**Anti-pattern:** Reusing Open’s `expandedData` shape or skipping presenter review for non-open tables.

## Mobile conditional render translation (systematic audit)

Treat mobile as **logic migration**:

- Build a short **matrix**: legacy surface → condition → user-visible outcome → location in `*HtmlTable.vue`.
- **Rules:** pair header/body `v-if` where the legacy table paired them; handle duplicated desktop + mobile paths; map `mc-table` responsive/hidden columns; do not trust stale comments—search `isSmallScreen` (and equivalents) on the **legacy** table. Validate **colspan / folded columns** when legacy folded extra fields into one column.
- **Tests:** Vitest toggles `isSmallScreen` via Pinia/app store where applicable; Cypress exercises **both** mobile and desktop when behavior diverges.

## Assessment rubric (legacy component vs `*HtmlTable.vue`)

When reviewing a migration (or pasting legacy `mc-table` code next to the new file), score quality against:

1. **Logic preserved** — sorting, selection, pagination, hover menus, copy/analytics, store updates, and **tab-specific** flows (e.g. disputed API sort refetch) still fire from the new template/composables.
2. **No unintended API/UX regressions** — props to child cells, disabled/cancelled row rules, and PDF/download behavior match the legacy slot markup.
3. **Tests** — targeted **unit** + **component** specs pass; extend coverage for new branches (mobile expand, cancelled rows, sort toggles) when the legacy template had them.
4. **Conditional UI** — desktop-only and mobile-only blocks (including **inline** tags in amount columns) still exist; **expanded** and **desktop error** rows follow the same `v-if` structure as the old table ([Open vs other tabs](#open-vs-other-tabs-wiring-differs)).
5. **Selectors and DOM** — `data-header-id`, `data-cy`, `data-test` align with [references/selector-mapping.md](references/selector-mapping.md); account for **virtualization** on Open when asserting row counts.

Use findings to update this skill and add a small test when you fix a gap.

## Iterating on skill quality (feedback loop)

1. **Capture** a gap (expanded data vs conditionals vs selectors vs styles).
2. **Classify** the failure mode.
3. **Edit** this skill + `references/*` with one minimal, testable rule.
4. **Add** a minimal failing-then-passing test in the app when possible.
5. Optionally re-run the same migration task with the updated skill (fixed commit, separate worktrees if agents write files).

Goal: each production or review finding becomes **one skill tweak + one test** where feasible.

## Scoped styles: drop unnecessary `:deep`

Legacy **`mc-table`** rendered inside **shadow DOM**, so `<style scoped>` often used **`:deep(...)`** to pierce internal header/cell markup. After migration, `<table>`, `<thead>`, `<th>`, `<tbody>`, `<td>`, and classes like `.mds-table__column--row-expander` are **normal elements in the same SFC** as the styles—Vue’s scoped transform already applies—**do not wrap** those selectors in `:deep()`.

- **Prefer** plain selectors, e.g. `.mds-table thead th`, `.mds-table tbody td`, `.{domain}-table--mobile .mds-table__column--row-expander`.
- **Keep `:deep` only** when styling **another component’s** template/slot content or third-party internals that scoped CSS cannot reach (rare for the table shell).
- When editing migrated `*HtmlTable.vue` files, remove leftover `:deep` that only targets **this** component’s native table markup—behavior should be unchanged; intent is clearer. Shipped tables may still contain **legacy `:deep(.mds-table …)`** from earlier migrations; remove when you touch the file, or track a dedicated cleanup.

### SCSS: `@use` paths for shared `invoice-table` styles

`@use` is resolved from the **`.vue` file’s directory** (Vite/Vue convention):

| SFC location | Typical `@use` |
|--------------|----------------|
| `src/components/FooHtmlTable.vue` | `@use "scoped-styles/invoice-table";` |
| `src/components/credits/CreditsHtmlTable.vue` | `@use "./credits-table";` and `@use "../scoped-styles/invoice-table";` (nested folders may add a **local** partial **plus** the shared invoice-table sheet) |

Copying a `<style>` block into a nested folder **without** adjusting the path breaks `npm run start` / Vite (“Can’t find stylesheet to import”). Run `npm run start` or `npm run build` after moves.

## Migration workflow (checklist)

1. **Inventory legacy behavior** and **parity:** sorting, pagination, selection, PDF links, copy buttons, tags, mobile layout, expanded rows, analytics (`copyButtonClicked`, etc.).
   - **Slot → `<td>` prop audit:** [Slot-to-cell prop audit (parity)](#slot-to-cell-prop-audit-parity)—especially shared cells (`DocumentReferenceCell`, copy blocks).
   - **Mc-table → HTML map:** [Quick map: mc-table features to HTML](#quick-map-mc-table-features-to-html).
   - **Open virtualization:** [Window virtualization (Open invoices)](#window-virtualization-open-invoices) if tests query row counts in the DOM.
   - **Mobile expanded / presenter parity:** [Mobile expanded row parity](#mobile-expanded-row-parity-production-lesson).
   - **Mobile vs desktop conditionals:** [Mobile conditional render translation](#mobile-conditional-render-translation-systematic-audit).
   - **Scoped styles:** prefer native table selectors **without** redundant `:deep` ([Scoped styles](#scoped-styles-drop-unnecessary-deep)).
2. **Match column `id`s** from the old config to **`data-header-id`** on existing invoice HtmlTables, **or** to **`data-column-id` / `data-cell-id`** when doing a scoped refresh per `html-table-components`. If you adopt the newer attributes, grep and update **unit, component, and e2e** selectors that still assume `data-header-id`.
3. **Implement `<table>`** with `scope="col"` on `<th>`, stable **row key** (`:key="row.id"`), and **row test id** (`:data-cy="row.id"`) where tests rely on it.
4. **Import MDS** web components actually used (`mc-checkbox`, `mc-button`, `mc-icon`, `mc-pagination`, `mc-c-copy-item`, …) — same as sibling tables.
5. **Styles:** reuse `@use "scoped-styles/invoice-table"` or established table SCSS (see [SCSS `@use` paths](#scss-use-paths-for-shared-invoice-table-styles)); align with `mds-table` modifiers from `mds-component-table-html`; avoid redundant `:deep` on native table markup ([Scoped styles](#scoped-styles-drop-unnecessary-deep)).
6. **Tests** — follow [references/test-structure.md](references/test-structure.md): **BDD `it` titles**, **feature-grouped `describe`** (Sorting, Selection, Pagination, …), **no root-level `it`**. Cover **mobile expanded parity** and **conditional parity** when the legacy table differed by viewport.
   - **Unit:** shallow/deep as appropriate; mock stores; assert sort/selection helpers; toggle `isSmallScreen` when testing responsive branches.
   - **Component (Cypress):** update selectors from slot-based assumptions to `th`/`td` + `data-*` / `data-cy` patterns used in the new template; add **Mobile expanded parity** (or equivalent) when needed.
   - **E2E:** search `tests/e2e` for `data-header-id`, `mc-table`, and table-specific step definitions. **Shipped invoice tables today** mostly use `th`/`td` **`data-header-id`**. If you move to **`data-column-id` / `data-cell-id`**, update steps to `th[data-column-id='…']` / `td[data-cell-id='…']` consistently. See [references/selector-mapping.md](references/selector-mapping.md) for DOM/e2e alignment with shipped tables.
7. **Parity pass:** totals, empty states, currency formatting, and mobile-only columns must match product expectations; document intentional differences in the PR.

## Do not (anti-patterns)

- Copy-paste an entire table from another tab without adjusting **variant**, **store**, and **column set**.
- Drop composables and reimplement selection/expansion logic ad hoc.
- Assume **global** uniqueness of `data-header-id` on `<td>` — that value repeats per column; use **`data-cell-id`** (see `html-table-components`) when you need a unique per-cell hook for new work.
- **Incomplete mobile expanded data** in presenters—users lose fields that only appeared in legacy mobile/expanded slots ([Mobile expanded row parity](#mobile-expanded-row-parity-production-lesson)).
- **Missing props on migrated slot content**—e.g. omitting **`DocumentReferenceCell` `:HBL`** when the legacy slot passed `documentReference` ([Slot-to-cell prop audit](#slot-to-cell-prop-audit-parity)).
- **Wrong or drifted mobile conditionals**—header/body `v-if` mismatched or breakpoints unlike legacy ([Mobile conditional render translation](#mobile-conditional-render-translation-systematic-audit)).
- **Flat or vague specs**—root-level `it` in component, view, or Cypress table specs, `describe("snapshots")` with non-BDD titles, or `describe` names about file mechanics instead of **functionality**; see [references/test-structure.md](references/test-structure.md).
- **Redundant `:deep` on native table markup** after migration ([Scoped styles](#scoped-styles-drop-unnecessary-deep)).
- **Blind `mc-table` prop → `.mds-table` modifier mapping** for presentation (hover, zebra, lines)—see [mc-table-only props vs plain HTML table + MDS](#mc-table-only-props-vs-plain-html-table--mds-no-blind-parity-rule); do not block merges on guessed equivalence to MDS foundations.

## After editing

Run targeted **unit** and **component** specs for the touched `*HtmlTable` and parent view; run relevant **e2e** if step definitions or selectors changed.
