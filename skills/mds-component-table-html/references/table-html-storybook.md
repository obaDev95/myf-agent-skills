# Table HTML — Storybook notes (MDS documentation, Jan 2026)

Source: [MDS Storybook — Table HTML documentation](https://mds.maersk.io/?path=/story/components-table-html--documentation). Class names and modifier behaviour are cross-checked against `@maersk-global/mds-foundations/scss/_table.scss` (MDS v2.170.x line of releases). The snippet below mirrors the documented example structure; capture it yourself from Storybook if you need to diff against a newer MDS version.

## When to use

- Prefer `mc-table` for fully componentized experiences; fall back to HTML tables plus `mds-table*` classes when integrating 3rd-party grids (TanStack Table, AG Grid) or server-rendered tables that must match MDS.
- Wrap every `<table>` in a `<div class="mds-table">` so border, radius, and scroll affordances render correctly.
- Load the SCSS once per bundle: `@use '@maersk-global/mds-foundations/scss/table';` (or import the prebuilt CSS bundle if your build already consumes `mds-foundations.scss`).

## Base markup (from Storybook documentation example)

```html
<body class="mds">
  <div class="mds-table mds-table--scrollable">
    <table>
      <thead>
        <tr>
          <th>Name</th>
          <th>Last port</th>
          <th class="mds-table__cell--number">Built (year)</th>
          <th class="mds-table__cell--number">Length (m)</th>
          <th class="mds-table__cell--number">Capacity (TEU)</th>
        </tr>
      </thead>
      <tbody>
        <!-- ship rows ... -->
      </tbody>
    </table>
  </div>
</body>
```

## Core tokens

- `.mds-table` sets border radius, background, header/body typography, and exposes `--row-border-radius` for trimming rounded cells when scrollbars appear.
- `.mds-table__subtext` (span inside `th`/`td`) forces block layout, small/medium typography, and neutral tone tokens.
- `.mds-table__cell--number` extends `--text-right` + `--tabular-figures` so numeric columns align visually.

## Size modifiers

| Class                | Notes                                                                                                               |
| -------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `.mds-table--small`  | tight padding, small typography, `@include mds-apply-expanded-row-padding('small')`.                                |
| `.mds-table--medium` | default body size, used when row expansion should match medium spacing.                                             |
| `.mds-table--large`  | generous padding, ensures checkboxes/buttons align flush; apply when tables appear in spacious cards or dashboards. |

## Row striping & separators

- `.mds-table--zebra-stripes` gives alternating row backgrounds; pair with `--disable-row-highlight-on-hover` to freeze colors.
- `.mds-table--zebra-stripes-with-expand` keeps zebra colors even while expanded rows are visible.
- Line controls: combine `.mds-table--horizontal-lines-{dashed|dotted|none}` with `.mds-table--vertical-lines-{solid|dashed|dotted|none}` to match grid density requirements.
- Border controls: `.mds-table--outer-border-{none|dashed|dotted|corners-square}` adjust the shell without touching internal dividers.

## Wrapping & alignment utilities

- `.mds-table--nowrap` prevents wrapping across the whole table; `.mds-table__cell--nowrap` scopes to specific cells.
- Vertical alignment helpers: `.mds-table--vertical-align-{top|baseline|bottom}` (table-level) or cell-level `.mds-table__cell--content-{top|center|bottom}`.
- Text alignment helpers: `.mds-table__cell--text-{center|right}` plus `.mds-table__cell--tabular-figures` for monospaced numbers.

## Interactive columns & states

- `.mds-table__column--row-selector` + `.mds-table__row-selector > mc-checkbox` adds a checkbox column with preset spacing tokens.
- `.mds_table__row--selected` paints the entire row with selection tokens—toggle this class when syncing selection with your data layer.
- `.mds-table__column--row-expander` reserves a column for `<mc-button icon="chevron-down" variant="text">`; pair with `.mds-table__expanded-row__trigger` so the icon rotates (adds `&--expanded` state).
- Expanded content sits in `<tr class="mds-table__expanded-row mds-table__expanded-row--visible">` or `--hidden` for collapsed rows.
- `.mds-table__child-row td` automatically dims typography for nested detail rows.

## Sticky + scroll behaviors

- `.mds-table--scrollable` applies `overflow:auto` to the wrapper and removes rounded corners nearest the scrollbar.
- `.mds-table__column--sticky` (use on any `th/td`) locks the column to the left with inset shadow separation.
- `.mds-table--header-sticky` (optionally paired with `--header-sticky-viewport`) pins header rows; set `--mds-table-header-sticky-top` from your layout (e.g., Tailwind `style="--mds-table-header-sticky-top: var(--topbar-height);"`).
- `.mds-table--footer` exposes `<tfoot>` styling, while `.mds-table--footer-sticky` keeps summary rows visible in long tables.

## Cell-level utilities

- Numeric emphasis: `.mds-table__cell--number` (right aligned + tabular figures) or mix `--text-right` + `--tabular-figures` separately.
- Content wrapping: drop helper spans with `.mds-table__subtext` for secondary lines.
- Sticky columns respect `.mds-table__column--sticky` plus `.mds-table--scrollable` to avoid clipped shadows.

## Captions & context

- Wrap tables that need a headline/legend in `.mds-table-and-caption`; append `.mds-table-caption` (plus `--small` / `--large`) for description blocks controlled by typography tokens.
- Notifications or helper copy can sit above the table via `<mc-notification>` (Storybook uses this for the “Foundational classes…” message).

## Implementation checklist

1. Import the foundations SCSS before your component styles; ensure `<body class="mds">` is set globally.
2. Wrap every `<table>` with `.mds-table`; add modifiers for scroll, sticky headers, zebra stripes, etc.
3. Use semantic HTML for headers/footers; only reach for utility classes when the layout requires them (numbers, nowrap, sticky, selection, expansion).
4. Keep interactive controls (checkboxes, expander buttons) as MDS components so focus rings and tokens stay consistent.
5. Document the class combinations you choose inside your Vue components to make maintenance easier.
