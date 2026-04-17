---
name: mds-component-table-html
description: >-
  Modifier and token catalogue for Maersk `.mds-table` HTML tables — sizes, dividers,
  zebra, sticky/scroll, alignment, row utilities, caption and subtext. Use as a reference
  when picking modifiers or resolving presentation questions; use `html-table-components`
  for the canonical SFC shape, and `invoice-html-table-migration` for the invoice-domain
  composables.
---

# MDS `.mds-table` modifier catalogue

This skill is a **reference** for `.mds-table` classes, tokens, and the a11y contract they imply. It does not prescribe a canonical SFC shape for this repo — **[html-table-components/SKILL.md](../html-table-components/SKILL.md)** does that. Load this skill when you need to pick a modifier or resolve a "which class does X" question; load the migration skills when you are writing or reviewing an `*HtmlTable.vue`.

---

## References

- Storybook Table HTML documentation (capture 13 Mar 2026) — rendered markup plus canonical `.mds-table` structure, sticky header guidance, and baseline HTML snippet.
- CSS foundations notes (`references/table-html-storybook.md`) — Storybook-aligned markup and a modifier catalogue derived from `@maersk-global/mds-foundations/scss/_table.scss` (alignment helpers, sticky column tokens, interactive row classes).
- Maersk Design System article "Table" (updated 13 Mar 2026) — best-practice narrative for alignment, overflow, expandable rows, pagination states, icon usage, and accessibility requirements.

---

## Workflow

1. **Load foundations**: import `@maersk-global/mds-foundations/scss/table` (or the foundations bundle that already forwards it) once per app shell and guarantee `<body class="mds">`. This exposes every `.mds-table*` modifier plus `--mds-table-header-sticky-top` without leaking duplicates.
2. **Wrap and label the table**: every `<table>` lives inside a `.mds-table` wrapper (a `<div>` in Storybook, a `<section>` in this repo's invoice tables). When the table needs surrounding copy, nest both inside `.mds-table-and-caption` and set `.mds-table-caption(--small|--large)`. Always pair the wrapper with a `<caption>` or `aria-labelledby` plus `role="region"` so screen readers hear the block description.
3. **Pick modifiers intentionally**: combine size (`--small|medium|large`), zebra (`--zebra-stripes` or `--zebra-stripes-with-expand`), divider (`--horizontal-lines-*`, `--vertical-lines-*`, `--outer-border-*`), and highlight (`--disable-row-highlight-on-hover`) tokens based on the data density described in the Maersk article. Avoid mutually exclusive mixes such as `--horizontal-lines-none` plus `--horizontal-lines-dashed`.
4. **Apply cell utilities**: numeric content uses `.mds-table__cell--number` (right alignment + tabular figures) or `.mds-table__cell--text-right` plus `.mds-table__cell--tabular-figures`. Secondary copy sits inside `.mds-table__subtext`. Use `.mds-table--nowrap` only when truncation is accompanied by tooltips per UX guidelines; otherwise scope wrapping via `.mds-table__cell--nowrap`.
5. **Plan scroll and sticky affordances**: add `mds-table--scrollable` whenever columns overflow. Enable `--header-sticky` or `--header-sticky-viewport` and set `style="--mds-table-header-sticky-top: <token>;"` so pinned headers clear the top bar. Footer totals use `--footer` plus `--footer-sticky` when context must remain visible. Sticky columns rely on `.mds-table__column--sticky` applied left-to-right.
6. **Wire interactive columns**: row selectors start with `.mds-table__column--row-selector` + `<mc-checkbox>`, and expanded detail rows use `.mds-table__column--row-expander`, `.mds-table__expanded-row__trigger`, and `.mds-table__expanded-row(--visible|--hidden)`. Keep triggers 44px min per `mds-ux-guidelines` and sync `aria-selected`/`aria-expanded` states with the row modifiers.
7. **Document accessibility evidence**: capture keyboard order, `aria-sort` updates, sticky behavior, and scroll-region labels in the story, QA notes, or visual diff. Reference the `mds-ux-guidelines` checklist so QA can validate WCAG 2.2 AA before merge.

---

## Design guidance (Maersk article, 13 Mar 2026)

- Prefer the HTML path when componentized behaviour is not needed or when integrating 3rd-party grids; in this repo, the migration away from `<mc-table>` makes the HTML path the default — see **[html-table-components/SKILL.md](../html-table-components/SKILL.md)**.
- Apply the "less is more" rule: keep essential columns only, collapse paired attributes with `.mds-table__subtext`, replace repetitive yes/no copy with icons or tags, and abbreviate units in the headers.
- Alignment: left-align text and non-countable numerals, right-align numbers, keep header alignment consistent with cell alignment to remove the need for vertical grid lines. Center columns only when content (icons) has equal width.
- Overflow: prioritize columns by importance, allow horizontal scroll with `mds-table--scrollable`, and use sticky headers / first columns to preserve context on narrow screens.
- Expandable rows: limit to one nested level, keep expanded content scannable, fall back to drawers or modals when supplemental data exceeds half the viewport.
- States and tone: provide explicit empty / loading / no-results / error treatments using captions or `mc-notification`; display dashes for missing values so users know data loaded; keep copy sentence case and optimistic per content tone guidelines.
- Accessibility: always include a caption or aria label, avoid empty cells, stick to semantic tables (not for layout), and switch to a grid pattern if the table holds many interactive widgets.

---

## Key attributes

- **Structure**: `.mds-table` wrapper + `<table>` skeleton, optionally wrapped by `.mds-table-and-caption` with `.mds-table-caption(--small|--large)` for contextual copy.
- **Sizing**: `mds-table--small`, `--medium`, `--large` adjust padding, typography, and expanded-row spacing.
- **Dividers & decoration**: zebra stripes (`--zebra-stripes`, `--zebra-stripes-with-expand`), line controls (`--horizontal-lines-{solid|dashed|dotted|none}`, `--vertical-lines-*`), shell controls (`--outer-border-{none|dashed|dotted|corners-square}`).
- **Scroll & sticky**: `mds-table--scrollable`, `--header-sticky`, `--header-sticky-viewport`, `--footer`, `--footer-sticky`, `.mds-table__column--sticky`, `--disable-row-highlight-on-hover`.
- **Alignment helpers**: `.mds-table--vertical-align-{top|baseline|bottom}` (table-wide), `.mds-table__cell--content-{top|center|bottom}` (per-cell), `.mds-table__cell--number`, `.mds-table__cell--text-{center|right}`, `.mds-table__cell--tabular-figures`.
- **Row utilities**: `.mds-table__column--row-selector`, `.mds-table__column--row-expander`, `.mds-table__expanded-row`, `.mds-table__child-row`, `.mds-table__row--selected`.
- **Context blocks**: `.mds-table__subtext` for secondary lines, `.mds-table-caption` for descriptive copy, `.mds-table--nowrap` / `.mds-table__cell--nowrap` for truncation scenarios that also surface tooltips.

---

## Events & interactions

- **Row selection**: pair `.mds-table__column--row-selector` with `<mc-checkbox>` and update `.mds-table__row--selected` plus `aria-selected` when the checkbox fires `change`. For select-all, reflect `indeterminate` state and describe the action via `aria-label`.
- **Expansion**: use `.mds-table__column--row-expander` with a text-variant `<mc-button icon="chevron-down">`. Toggle `.mds-table__expanded-row--visible` and mirror that state with `aria-expanded` on the trigger.
- **Sorting**: clickable headers should announce `aria-sort="ascending|descending|none"`, show an icon or text hint, and update both the DOM order and the attribute together.
- **Scroll regions**: when `mds-table--scrollable` is active, add `role="region"` plus `aria-labelledby` or `aria-label`, and set `--mds-table-header-sticky-top` to the top-bar token so sticky headers never overlap navigation.
- **3rd-party grids**: store the modifier set (e.g., `['mds-table--scrollable','mds-table--zebra-stripes']`) in a config module and map your grid's column metadata to the appropriate `.mds-table__cell--*` classes so audits stay reproducible.
- **Icon and badge placement**: center icon-only columns using `.mds-table__cell--text-center`, rely on `mc-icon` or `mc-tag` for statuses, and add visually hidden text (or `aria-label`) when symbols replace copy.

---

## Usage checklist

- Stylesheet imported once, `body` carries `mds`, and the `.mds-table` wrapper has no inline bespoke borders.
- Each table declares a caption or `aria-labelledby`, uses `scope="col"` / `scope="row"` or `id` / `headers`.
- Numeric and right-aligned columns use `.mds-table__cell--number` or related helpers; missing data renders as `--` rather than empty cells.
- Scrollable or sticky tables expose `role="region"`, set `--mds-table-header-sticky-top`, and keep focus order matching visual order.
- Selection / expansion triggers update both visual modifiers and `aria-selected` / `aria-expanded`; interactive rows stay within the 44px touch target minimum.
- Empty / loading / no-result / error states are implemented with captions or `mc-notification`, not ad-hoc copy.
- Accessibility evidence (keyboard walkthrough, screen-reader output, sticky-behavior screenshots) is logged alongside test runs before handoff.

---

## Where the canonical shape lives

For a working scaffold (region wrapper, `<thead>` / `<tbody>` layout, selection + expander column order, mobile expanded row, desktop notification row, sort button shape, row expander shape), use **[html-table-components/SKILL.md](../html-table-components/SKILL.md)**. This skill's role is to answer "which modifier does X" — not to re-teach the SFC scaffold.
