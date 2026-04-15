---
name: mds-component-table-html
description: Shape Maersk-branded HTML tables with `.mds-table` classes whenever `<mc-table>` is not feasible; this skill covers modifiers, Vue 3 `<script setup>` wrappers, Tailwind layout, and QA evidence so teams know when to trigger the CSS foundations path.
---

## References
- Storybook Table HTML documentation (capture 13 Mar 2026) — rendered markup plus canonical `.mds-table` structure, sticky header guidance, and baseline HTML snippet for copy-paste scaffolding.
- CSS foundations snapshot (`references/table-html-storybook.md`) — Playwright capture of the `@maersk-global/mds-foundations/scss/_table.scss` modifier catalogue, alignment helpers, sticky column tokens, and interactive row classes.
- Maersk Design System article “Table” (updated 13 Mar 2026) — best-practice narrative for alignment, overflow, expandable rows, pagination states, icon usage, and accessibility requirements across HTML and component implementations.

## Workflow
1. **Load foundations**: import `@maersk-global/mds-foundations/scss/table` (or the foundations bundle that already forwards it) once per app shell and guarantee `<body class="mds">`. This exposes every `.mds-table*` modifier plus `--mds-table-header-sticky-top` without leaking duplicates.
2. **Wrap and label the table**: every `<table>` lives inside `<div class="mds-table [modifiers]">`. When the table needs surrounding copy, nest both inside `.mds-table-and-caption` and set `.mds-table-caption(--small|--large)` to mirror spec tone. Always pair the wrapper with either a `<caption>` or `aria-labelledby` plus `role="region"` so screen readers hear the block description.
3. **Pick modifiers intentionally**: combine size (`--small|medium|large`), zebra (`--zebra-stripes` or `--zebra-stripes-with-expand`), divider (`--horizontal-lines-*`, `--vertical-lines-*`, `--outer-border-*`), and highlight (`--disable-row-highlight-on-hover`) tokens based on the data density described in the Maersk article. Avoid mutually exclusive mixes such as `--horizontal-lines-none` plus `--horizontal-lines-dashed`.
4. **Apply cell utilities**: numeric content uses `.mds-table__cell--number` (right alignment + tabular figures) or `.mds-table__cell--text-right` plus `.mds-table__cell--tabular-figures`. Secondary copy sits inside `.mds-table__subtext`. Use `.mds-table--nowrap` only when truncation is accompanied by tooltips per UX guidelines; otherwise scope wrapping via `.mds-table__cell--nowrap`.
5. **Plan scroll and sticky affordances**: add `mds-table--scrollable` whenever columns overflow. Enable `--header-sticky` or `--header-sticky-viewport` and set `style="--mds-table-header-sticky-top: <token>;"` so pinned headers clear the top bar. Footer totals use `--footer` plus `--footer-sticky` when context must remain visible. Sticky columns rely on `.mds-table__column--sticky` applied left-to-right.
6. **Wire interactive columns**: row selectors start with `.mds-table__column--row-selector` + `<mc-checkbox>`, and expanded detail rows use `.mds-table__column--row-expander`, `.mds-table__expanded-row__trigger`, and `.mds-table__expanded-row(--visible|--hidden)`. Keep triggers 44px min per `mds-ux-guidelines` and sync `aria-selected`/`aria-expanded` states with the row modifiers.
7. **Pair with Vue + Tailwind**: build data/state in `<script setup lang="ts">` using `ref`, `computed`, and `MaybeRef` friendly helpers. Import only the MDS components you render (`@maersk-global/mds-components-core-checkbox`, buttons, icons) and keep Tailwind utilities to outer layout containers (`gap-*`, `max-h-*`). Inside the `.mds-table` wrapper, rely on design tokens rather than ad-hoc classes.
8. **Document accessibility evidence**: capture keyboard order, `aria-sort` updates, sticky behavior, and scroll-region labels in the story, QA notes, or visual diff. Reference the `mds-ux-guidelines` checklist so QA can validate WCAG 2.2 AA before merge.

## Design guidance (Maersk article, 13 Mar 2026)
- Prefer the HTML path for legacy SSR pages, 3rd-party grid adapters (TanStack, AG Grid), or hard-server-rendered exports; otherwise use `<mc-table>` for built-in pagination, selection, and analytics.
- Apply the “less is more” rule: keep essential columns only, collapse paired attributes with `.mds-table__subtext`, replace repetitive yes/no copy with icons or tags, and abbreviate units in the headers.
- Alignment rules: left-align text and non-countable numerals, right-align numbers, and keep header alignment consistent with cell alignment to remove the need for vertical grid lines. Center columns only when content (icons) has equal width.
- Overflow strategy: prioritize columns by importance, allow horizontal scroll with `mds-table--scrollable`, and use sticky headers/first columns to preserve context on narrow screens. Combine columns cautiously so scanning remains possible.
- Expandable rows: limit to one nested level, keep expanded content scannable (headings, lists, short paragraphs), and fall back to drawers or modals when supplemental data exceeds half the viewport.
- States and tone: provide explicit empty/loading/no-results/error treatments using captions or `mc-notification`, display dashes for missing values so users know data loaded, and keep copy sentence case and optimistic per content tone guidelines.
- Accessibility: always include a caption or aria label, avoid empty cells, stick to semantic tables (not for layout), and switch to a grid pattern if the table holds many interactive widgets so keyboard users can navigate with arrows instead of tabbing through every control.

## Key attributes
- **Structure**: `.mds-table` wrapper + `<table>` skeleton, optionally wrapped by `.mds-table-and-caption` with `.mds-table-caption(--small|--large)` for contextual copy.
- **Sizing**: `mds-table--small`, `--medium`, `--large` adjust padding, typography, and expanded-row spacing to match compact dashboards or spacious analytical pages.
- **Dividers & decoration**: zebra stripes (`--zebra-stripes`, `--zebra-stripes-with-expand`), line controls (`--horizontal-lines-{solid|dashed|dotted|none}`, `--vertical-lines-*`), and shell controls (`--outer-border-{none|dashed|dotted|corners-square}`) let you tune density without custom CSS.
- **Scroll & sticky**: `mds-table--scrollable`, `--header-sticky`, `--header-sticky-viewport`, `--footer`, `--footer-sticky`, `.mds-table__column--sticky`, and `--disable-row-highlight-on-hover` cover virtually every overflow pattern described in the Maersk article.
- **Alignment helpers**: `.mds-table--vertical-align-{top|baseline|bottom}` for table-wide control, `.mds-table__cell--content-{top|center|bottom}` for per-cell overrides, `.mds-table__cell--number`, `.mds-table__cell--text-{center|right}`, and `.mds-table__cell--tabular-figures` for numeric polish.
- **Row utilities**: `.mds-table__column--row-selector`, `.mds-table__column--row-expander`, `.mds-table__expanded-row`, `.mds-table__child-row`, and `.mds-table__row--selected` align selectors, expander triggers, nested rows, and selection tokens with the visual spec.
- **Context blocks**: `.mds-table__subtext` for secondary lines, `.mds-table-caption` for descriptive copy, and `.mds-table--nowrap` / `.mds-table__cell--nowrap` for truncation scenarios that also surface tooltips.

## Events & interactions
- **Row selection**: pair `.mds-table__column--row-selector` with `<mc-checkbox>` and update `.mds-table__row--selected` plus `aria-selected` when the checkbox fires `change`. For select-all, reflect `indeterminate` state and describe the action via `aria-label` so assistive tech hears it.
- **Expansion**: use `.mds-table__column--row-expander` with a text-variant `<mc-button icon="chevron-down">`. Toggle `.mds-table__expanded-row--visible` and mirror that state with `aria-expanded` on the trigger. Limit expansion to one open row when space is constrained, otherwise document how multiple expansions stay within 50% of the viewport.
- **Sorting**: clickable headers should announce `aria-sort="ascending|descending|none"`, show an icon or text hint per the design article, and update both the DOM order and the attribute together to keep AT and visuals in sync.
- **Scroll regions**: when `mds-table--scrollable` is active, add `role="region"` plus `aria-labelledby` or `aria-label`, and set `--mds-table-header-sticky-top` to the top-bar token so sticky headers never overlap navigation.
- **3rd-party grids**: store the modifier set (e.g., `['mds-table--scrollable','mds-table--zebra-stripes']`) in a config module and map your grid’s column metadata to the appropriate `.mds-table__cell--*` classes so audits stay reproducible.
- **Icon and badge placement**: center icon-only columns using `.mds-table__cell--text-center`, rely on `mc-icon` or `mc-tag` for statuses, and add visually hidden text (or `aria-label`) when symbols replace copy so tone stays human per the UX guidelines.

## Usage checklist
- Stylesheet imported once, `body` carries `mds`, and `.mds-table` wraps every `<table>` with no inline bespoke borders.
- Each table declares a caption or `aria-labelledby`, uses `scope="col"`/`scope="row"` or `id`/`headers`, and documents why the HTML path was chosen instead of `<mc-table>`.
- Numeric and right-aligned columns use `.mds-table__cell--number` or related helpers; missing data renders as `--` (or similar) rather than empty cells, matching the Maersk article.
- Scrollable or sticky tables expose `role="region"`, set `--mds-table-header-sticky-top`, and keep focus order matching visual order per `mds-ux-guidelines`.
- Selection/expansion triggers update both visual modifiers and `aria-selected` / `aria-expanded`, and interactive rows stay within the 44px touch target minimum.
- Zebra, divider, and highlight modifiers are documented in Storybook notes so QA can diff them when tokens change.
- Empty/loading/no-result/error states are implemented with captions or `mc-notification`, not ad-hoc copy.
- Accessibility evidence (keyboard walkthrough, screen-reader output, sticky-behavior screenshots) is logged alongside test runs before handoff.

## Vue example
```vue
<script setup lang="ts">
import { computed, ref } from 'vue'
import '@maersk-global/mds-components-core-checkbox'
import '@/styles/mds.scss'

interface VesselRow {
  id: string
  name: string
  port: string
  builtYear: number
  lengthM: number
  teu: number
}

const rows = ref<VesselRow[]>([
  { id: 'mad', name: 'Madrid Maersk', port: 'Shanghai', builtYear: 2017, lengthM: 399, teu: 19630 },
  { id: 'mary', name: 'Mary Maersk', port: 'Busan', builtYear: 2013, lengthM: 399, teu: 18270 },
  { id: 'emm', name: 'Emma Maersk', port: 'Los Angeles', builtYear: 2006, lengthM: 398, teu: 15550 },
  { id: 'ger', name: 'Gerner Maersk', port: 'Rotterdam', builtYear: 2008, lengthM: 367, teu: 9038 },
  { id: 'sv', name: 'Svendborg Maersk', port: 'Manila', builtYear: 1998, lengthM: 347, teu: 8160 },
])

const selectedIds = ref<Set<string>>(new Set())
const allSelected = computed(
  () => rows.value.length > 0 && selectedIds.value.size === rows.value.length,
)
const partiallySelected = computed(
  () => selectedIds.value.size > 0 && !allSelected.value,
)
const toggleRow = (id: string) => {
  const next = new Set(selectedIds.value)
  next.has(id) ? next.delete(id) : next.add(id)
  selectedIds.value = next
}
const toggleAll = () => {
  selectedIds.value = allSelected.value
    ? new Set()
    : new Set(rows.value.map((row) => row.id))
}

const sortDirection = ref<'ascending' | 'descending'>('descending')
const sortedRows = computed(() => {
  const dir = sortDirection.value === 'ascending' ? 1 : -1
  return [...rows.value].sort((a, b) => (a.teu - b.teu) * dir)
})
const toggleCapacitySort = () => {
  sortDirection.value = sortDirection.value === 'ascending' ? 'descending' : 'ascending'
}

const tableLabelId = 'fleet-table-caption'
</script>

<template>
  <div class="space-y-4">
    <div class="mds-table-and-caption">
      <div
        class="mds-table mds-table--scrollable mds-table--zebra-stripes mds-table--header-sticky max-h-[28rem]"
        role="region"
        :aria-labelledby="tableLabelId"
        style="--mds-table-header-sticky-top: 3.5rem;"
      >
        <table>
          <caption
            :id="tableLabelId"
            class="mds-table-caption mds-table-caption--small"
          >
            Fleet readiness by capacity
          </caption>
          <thead>
            <tr>
              <th scope="col" class="mds-table__column--row-selector">
                <mc-checkbox
                  aria-label="Select all vessels"
                  :checked="allSelected"
                  :indeterminate="partiallySelected"
                  @change="toggleAll"
                />
              </th>
              <th scope="col">Name</th>
              <th scope="col">Last port</th>
              <th scope="col" class="mds-table__cell--number">Built (year)</th>
              <th scope="col" class="mds-table__cell--number">Length (m)</th>
              <th
                scope="col"
                class="mds-table__cell--number"
                :aria-sort="sortDirection"
              >
                <button type="button" @click="toggleCapacitySort">
                  Capacity (TEU)
                </button>
              </th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="row in sortedRows"
              :key="row.id"
              :class="{ 'mds-table__row--selected': selectedIds.has(row.id) }"
              :aria-selected="selectedIds.has(row.id)"
            >
              <td class="mds-table__column--row-selector">
                <mc-checkbox
                  :aria-label="`Select ${row.name}`"
                  :checked="selectedIds.has(row.id)"
                  @change="() => toggleRow(row.id)"
                />
              </td>
              <th scope="row">
                {{ row.name }}
                <span class="mds-table__subtext">ETA 48h</span>
              </th>
              <td>{{ row.port }}</td>
              <td class="mds-table__cell--number">{{ row.builtYear }}</td>
              <td class="mds-table__cell--number">{{ row.lengthM }}</td>
              <td class="mds-table__cell--number">
                {{ row.teu.toLocaleString() }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>
```

### Notes
- Keep Tailwind utilities outside the `.mds-table` construct (wrappers, spacing, max heights) and lean on MDS modifiers for anything inside the table so future token releases do not clash with local CSS.
- When integrating TanStack/AgGrid, create a `tableHtmlConfig.ts` that maps column IDs to `.mds-table__cell--*` helpers and stores modifier arrays, making audits and code reviews trivial.
- Record accessibility results (keyboard run, screen-reader narration of captions/sort, sticky behavior screenshots) alongside Storybook stories so QA has traceable evidence per `mds-ux-guidelines`.
