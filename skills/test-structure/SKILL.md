---
name: test-structure
description: >-
  BDD-style test titles and feature-grouped describe blocks for Vitest and Cypress in
  Vue projects. Use when structuring or reviewing specs (tables, views, components),
  enforcing no root-level it(), or when the user asks for consistent test organization
  across a feature area.
---

# Test structure: BDD titles and feature-grouped `describe` blocks

Conventions for **Vitest** (unit/component) and **Cypress** (component/e2e) specs. Applies to any domain—invoice tables, estatements, refunds, etc.

## Reference

- [references/test-structure.md](references/test-structure.md) — mandatory rules, feature groups, and mobile-conditional testing notes.

## Pairing with domain skills

When migrating or refactoring a specific feature (e.g. HTML tables), follow the **domain skill** for parity, selectors, and composables—use **this skill** only for **how** tests are grouped and titled.
