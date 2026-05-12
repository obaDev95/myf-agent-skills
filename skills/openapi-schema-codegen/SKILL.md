---
name: openapi-schema-codegen
description: >-
  Update OpenAPI contracts in schemas/, regenerate TypeScript clients into src/auto/api
  with npm run codegen-*, then align wrappers, interfaces, dev mock handlers (/myfinance/api),
  unit mocks, and Cypress. Use when contract-backed HTTP shapes change; not for ad-hoc
  fetch layers with no schema. Covers schema→script map, swagger-typescript-api flag drift,
  and post-codegen verification (typecheck, tests).
---

# OpenAPI schema → TypeScript codegen (ui-myfinance)

## Intent

When the **HTTP contract** for a Maersk My Finance integration changes, the source of truth in this repository is the **OpenAPI file under `schemas/`**. Agents and developers must:

1. **Edit the schema** (or replace it with an authoritative export from API/platform teams).
2. **Regenerate** the matching client under `src/auto/api/` using the **correct** `npm run codegen-*` script.
3. **Fix downstream TypeScript and tests** — generated output is not the whole story; wrappers, interfaces, mocks, and intercepts often need updates.

**Never hand-edit files under `src/auto/api/`** — they are overwritten by `swagger-typescript-api` and are explicitly out of bounds in `README.md` and `.github/copilot-instructions.md`.

### When to use this skill

- Request/response body, query parameters, headers documented in OpenAPI, or **operationId** / path changes for an API that already has a row in the **schema → script** table below.
- New paths or schemas in an **existing** `schemas/*` file (then regenerate that file’s script only).
- Adding a **new** contract file and `codegen-*` script (see **Adding a new API**).

### Out of scope (do not force through this playbook)

- Endpoints implemented only via **manual** `fetch`/axios in `src/lib/api/` (or elsewhere) with **no** matching file in `schemas/` — extend or add OpenAPI first, or treat as a separate integration task.
- UI-only refactors that do not change the HTTP contract.
- Changing **`swagger-typescript-api` CLI shape** on existing scripts to “look consistent” — follow the **Command-shape quirks** section instead.

Human-readable tables and policy also live in **`README.md`** (search for **“When to re-run codegen”**).

---

## What the minimal instruction leaves out

| Gap | Why it matters |
|-----|----------------|
| **Which `codegen-*` script?** | There is **no** `codegen-all`. Each schema maps to **one** npm script. Running the wrong one regenerates the wrong module (e.g. `invoices` vs `invoices-v2`). If you touched **two** schemas, run **two** matching scripts — never “all scripts”. |
| **`schemas/error.yml`** | `schemas/35-customer-cdt-oas.yaml` uses `$ref: ../schemas/error.yml` for shared error shapes. Customer/error model changes may require editing **both** files, then `npm run codegen-customer`. |
| **`src/lib/api/` and `src/lib/api.ts`** | `.github/copilot-instructions.md`: **wrap HTTP usage** in `src/lib/api/` (and the facade in `lib/api.ts`). Regeneration can rename types, enums, or `Api` methods — wrappers must compile and still send correct headers/paths. |
| **`src/interfaces/**`** | Many interfaces **import types from `@/auto/api/*`**. A response shape change can cascade into interface files and components. |
| **Root `mock/*.ts` (Vite mock server)** | `vite.config.mts` mounts **`vite-plugin-mock-server`** with `urlPrefixes: ["/myfinance/api"]`. Handlers in **`mock/`** mirror real routes for local dev; **path, method, or JSON body** changes need handler updates even when TypeScript still compiles. |
| **`mock/` data modules and `tests/unit/mock/`** | Fixtures typed against generated models break when properties or enum members change. |
| **Cypress / `cy.intercept`** | E2E and component tests match **URL paths and query strings**, not TypeScript types. Path or query contract changes need intercept + fixture updates even when codegen succeeds. |
| **Tooling** | `eslint.config.js` **ignores** `src/auto/**`. Root `tsconfig.json` **excludes** `src/auto/*`. Mistakes inside generated files are easy to miss — **regenerate from schema** instead of patching generated code. |
| **Tracked output** | `src/auto/api/*.ts` is **committed**. PRs should include **schema + regenerated TS** together so reviewers see the full contract diff. |
| **CLI flag drift** | Not every script uses identical `swagger-typescript-api` flags (e.g. `--union-enums` is on some scripts and off others). **Do not “normalize”** flags across scripts without an explicit team decision — output shape (string unions vs enums) can change widely. |

---

## Prerequisites

- Run commands from the **repository root**.
- Tool: **`swagger-typescript-api`** (devDependency; invoked via `npx` in scripts).
- After any change that affects app code: **`npm run typecheck`** and targeted **`npm run test`** / **`npm run test:unit`** (or domain-specific tests).

---

## Step 1 — Identify the schema file

All contract YAML/YML files live in **`schemas/`**. There is **no** bundling script; each file listed below is an independent **codegen entrypoint** (except **`error.yml`**, which is a **fragment** included by reference — currently from `35-customer-cdt-oas.yaml` only).

Extensions vary (`.yaml` vs `.yml`) — use the **exact path** from the table below when editing.

---

## Step 2 — Edit the OpenAPI document

- Apply **additive** changes (new fields with clear optionality) when the backend supports it, to minimise breaking UI assumptions.
- For **breaking** renames or removed properties, plan a **grep-driven** follow-up across `src/`, `mock/`, and `tests/` for old names.
- Keep **`openapi`**, **`info`**, **`servers`**, **`paths`**, and **`components/schemas`** consistent with what the gateway or service actually exposes.
- If `$ref` points to **`schemas/error.yml`**, update that fragment when shared error models change, then regenerate **customer** (see mapping table).

Validate locally if you use an OpenAPI validator in your editor; the repo does not run a schema lint in pre-commit.

---

## Step 3 — Run the matching codegen script

From the project root:

```bash
npm run codegen-<name>
```

Use this **schema → script → output module** map (aligned with `package.json` and `README.md`):

| npm script | Schema file | Generated file (`-n` → `src/auto/api/<name>.ts`) |
|------------|-------------|-----------------------------------------------------|
| `codegen-invoices` | `schemas/myfinance-invoices-API.v1.yml` | `invoices.ts` |
| `codegen-invoices-v2` | `schemas/myfinance-invoices-API.v2.yaml` | `invoices-v2.ts` |
| `codegen-proof-of-payment` | `schemas/myfinance-submit-proof-of-payment-API.v1.yaml` | `proof-of-payment.ts` |
| `codegen-payment-availability` | `schemas/PNC_PaymentAvailability-API.v1.yaml` | `payment-availability.ts` |
| `codegen-bank-profiles` | `schemas/PNC_BankProfiles-API.v1.yaml` | `bank-profiles.ts` |
| `codegen-export-documents` | `schemas/myfinance-export-documents-API.v1.yml` | `export-documents.ts` |
| `codegen-refund-request` | `schemas/myfinance-refund-request-API.v1.yaml` | `refund-request.ts` |
| `codegen-estatements` | `schemas/myfinance-estatements-API.v1.yml` | `estatements.ts` |
| `codegen-workflows` | `schemas/myfinance-workflows-API.v1.yaml` | `workflows.ts` |
| `codegen-customer` | `schemas/35-customer-cdt-oas.yaml` (+ `schemas/error.yml` when applicable) | `customer.ts` |
| `codegen-ai-service` | `schemas/myfinance-ai-service.yaml` | `pdf-summary.ts` |
| `codegen-notification-subscriptions` | `schemas/notifications-subscriptions-prod-oas.yaml` | `notification-subscriptions.ts` |

**Imports** in application code use the path alias, for example:

```ts
import { Api as InvoicesApi } from "@/auto/api/invoices";
import type { Invoice } from "@/auto/api/invoices-v2";
```

There is **no** barrel `index.ts` in `src/auto/api/`.

### Command-shape quirks (copy the sibling script; do not “tidy”)

In `package.json`, scripts are not byte-identical:

- **`codegen-invoices`** and **`codegen-invoices-v2`** call `swagger-typescript-api generate -p …` (explicit `generate` subcommand).
- **`codegen-ai-service`** uses `swagger-typescript-api generate --path ./schemas/…` (`--path` instead of `-p`).
- Most other scripts use `npx swagger-typescript-api -o … -p … -n …` without spelling `generate`.

All of these invoke the same tool; when **adding** a new script, mirror the **closest existing** API (especially invoices vs non-invoices). Do not rewrite existing commands to a single style unless the team owns a migration.

### Flag differences between scripts (do not “fix” casually)

Shared flags across scripts include `--sort-types --extract-request-params --extract-enums --add-readonly`.

**`--union-enums`** is enabled on: proof-of-payment, payment-availability, bank-profiles, estatements, workflows, customer, ai-service (pdf-summary), notification-subscriptions.

It is **disabled** on: invoices, invoices-v2, export-documents, refund-request.

Changing `--union-enums` on an existing script **changes how enums surface in TypeScript** and can create a large diff and widespread call-site churn. Treat flag changes as a **deliberate migration**, not a drive-by cleanup.

---

## Step 4 — Review the generated diff

Open `src/auto/api/<module>.ts` and confirm:

- Only **expected** types, operations, and paths changed.
- No accidental **operationId** or path renames (these rename `Api` class methods and break wrappers).
- New enums or unions appear where the UI must branch.

Generated files start with a **swagger-typescript-api** banner and typically include `// @ts-nocheck` — that is **expected**.

---

## Step 5 — Post-codegen checklist (mandatory)

Work through these in order for the APIs you touched:

1. **`npm run typecheck`** — fixes missing imports, renamed symbols, and stricter typings in **application** code (wrappers, stores, components).
2. **`src/lib/api.ts` and `src/lib/api/*.ts`** — update method names, request/response types, header contracts, and base URLs if the generated `Api` surface changed.
3. **`src/interfaces/**`** — update types that compose or re-export generated models.
4. **Mocks** — update root **`mock/*.ts`** handlers (Vite mock server, `/myfinance/api`) when routes or JSON bodies change; align **`tests/unit/mock/*.ts`** and other fixtures with new shapes; run unit tests that hit those mocks.
5. **Tests that `vi.mock("@/auto/api/...")`** — update mocks when class constructors or methods change (`grep` for the module name under `tests/`).
6. **Cypress `cy.intercept` and fixtures** — if paths, query keys, or JSON bodies in **real traffic** changed, update `tests/e2e/` (and component tests under `tests/component/` if they intercept the same routes).
7. **`grep` for removed enum members or renamed types** across `src/`, `mock/`, `tests/` to catch stringly-typed usage.

Run **`npm run test:unit`** (or a narrower Vitest filter) before opening a PR.

### Finding blast radius after a contract change

From the repo root, use the **output `-n` name** (e.g. `invoices-v2`, `customer`) as the module slug:

```bash
rg "@/auto/api/<slug>" src mock tests
rg "auto/api/<slug>" src mock tests
rg "<slug>" src/lib/api src/interfaces --glob "*.ts"
```

Also search for **operation names** or **path segments** if only part of the API moved.

---

## Troubleshooting

| Symptom | Likely cause | What to do |
|--------|----------------|-------------|
| `swagger-typescript-api` parse / resolver error | Invalid YAML or a broken `$ref` | Fix the schema; for customer, confirm **`schemas/error.yml`** exists and matches `$ref: ../schemas/error.yml` from `schemas/35-customer-cdt-oas.yaml`. |
| Huge unexpected diff in `src/auto/api/*.ts` | Accidental **path** / **operationId** rename, or edited generated file | Revert hand-edits to generated output; restore intended IDs in YAML; regenerate. |
| `npm run typecheck` clean but wrong runtime behaviour | Cypress intercepts, Vite **`mock/`** handlers, or string literals out of TS’s view | Grep route strings and response fixtures outside `src/`. |

---

## Adding a **new** API (not just updating responses)

1. Add the new **OpenAPI file** under `schemas/`.
2. Add a new **`codegen-*` script** in `package.json` following the same **`npx swagger-typescript-api`** pattern as sibling scripts (match **`--union-enums`** usage to similar APIs — enums vs string unions — or align with platform guidance).
3. Run the new script once; commit **`schemas/...`**, **`package.json`**, and **`src/auto/api/<new>.ts`**.
4. Implement **`src/lib/api/<wrapper>.ts`** (and wire into stores/composables per `.github/copilot-instructions.md`: schema → codegen → **wrapper** → store → telemetry).

---

## Anti-patterns

- **Editing `src/auto/api/*.ts` by hand** — changes are lost on the next codegen and bypass review of the real contract.
- **Running every `codegen-*` script** “just in case” — creates noisy unrelated diffs; run **only** the script(s) for schemas you actually changed.
- **Assuming ESLint will catch issues in `src/auto`** — it will not; rely on **typecheck** and **tests** for regressions.
- **Forgetting invoices v1 vs v2** — they are separate schemas and outputs; pick the one that matches the backend version your feature uses.

---

## Quick reference: one-liner workflow

1. Edit **`schemas/<file>`** (and **`schemas/error.yml`** if customer refs need it).  
2. Run **`npm run codegen-<name>`** from the mapping table.  
3. Review **`src/auto/api/<module>.ts`**.  
4. Update **`src/lib/api*`**, **`src/interfaces/**`**, **mocks**, **intercepts** as needed.  
5. Run **`npm run typecheck`** and **`npm run test:unit`** (or targeted tests).  
6. Commit **schema + generated client + application fixes** in a coherent change set.

For product architecture beyond codegen, see **`.github/copilot-instructions.md`** (wrapper-only usage of generated clients, store/telemetry flow).

**Commit message hint:** contract sync is usually **`chore(api): …`** or **`chore:`** with a short description of the schema or consumer area; keep schema and generated `src/auto/api/*.ts` in the **same** change set as the dependent wrapper/test fixes when possible.
