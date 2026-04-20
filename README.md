# myf-agent-skills

Agent skills for **ui-myfinance** and related workflows. Each skill lives under [`skills/<name>/`](skills/) with a `SKILL.md` entry point and optional `references/` files.

## Skills

| Skill | Summary |
|-------|---------|
| [`html-table-components`](skills/html-table-components/SKILL.md) | `<mc-table>` → semantic HTML `<table>` with `.mds-table`; shared SCSS and parity rules. |
| [`invoice-html-table-migration`](skills/invoice-html-table-migration/SKILL.md) | Per-tab migration playbook and selector mapping. |
| [`mc-table-legacy-audit`](skills/mc-table-legacy-audit/SKILL.md) | Pre-migration audit for legacy mc-table usage. |
| [`mds-component-table-html`](skills/mds-component-table-html/SKILL.md) | `.mds-table` modifier catalogue and a11y reference. |
| [`migration-pr-review`](skills/migration-pr-review/SKILL.md) | Reviewer checklist for HTML table migrations. |
| [`shared-table-scss-refactor`](skills/shared-table-scss-refactor/SKILL.md) | Shared SCSS partials for invoice tables. |
| [`test-structure`](skills/test-structure/SKILL.md) | BDD-style tests and feature-grouped `describe` blocks. |
| [`e-invoice-country-download`](skills/e-invoice-country-download/SKILL.md) | Enable e-invoice downloads by country or business area (`configItems`, `DownloadMenu`, mocks, tests). |

## ui-myfinance layout

In a full [ui-myfinance](https://github.com/Maersk-Global/ui-myfinance) checkout, mirror these skills under `myf-agent-skills/` (and optionally under `.cursor/skills/`) so agents can load them without depending on a single branch.
