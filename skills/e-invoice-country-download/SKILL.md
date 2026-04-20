---
name: e-invoice-country-download
description: >-
  Enables country- or business-area-based e-invoice downloads in ui-myfinance: configItems
  e-invoice maps, DownloadMenu routing (E_INVOICE_PREFIXES, getEInvoiceData), download
  extension logic, optional Vite mock JSON, and Vitest/Cypress tests. Covers both reusing
  an existing DocumentTypeEnum and adding a new enum via OpenAPI codegen. Use when
  enabling e-invoice for a country, debugging menu visibility, documentType, or file
  extension issues.
---

# E-invoice country download (ui-myfinance)

## When this skill applies

- Adding a **new** country or operating model where customers should see **Download e-invoice** and receive the correct file type.
- Fixing **missing menu item**, **wrong `documentType`**, **wrong extension** (json/xml/pdf/zip), or **bulk zip naming** for e-invoices.
- Coordinating **backend contract** (`DocumentTypeEnum`) with **frontend config** and **tests**.

## Mental model

1. **Eligibility to show the e-invoice action** is decided in `DownloadMenu.vue`: allowed tabs, customer country and/or business-area prefix, and invoice flags from the API (`hasJson`, `hasXML`, `hasEInvoice`, statuses).
2. **Which API document type and file extension to use** is split between `configItems.eInvoiceConfig`, `getDocumentType()` in `DownloadMenu.vue`, and `getExtension()` in `download.utility.ts`.
3. **`DocumentTypeEnum`** comes from `@/auto/api/export-documents` (generated). **Do not hand-edit** `src/auto/api/export-documents.ts`.

Read file-level detail in [references/flow-and-files.md](references/flow-and-files.md), including the **MYF-4368 (Belgium)** parity checklist.

---

## Decision: country-driven vs business-area-driven

| Situation | What to update |
|-----------|----------------|
| E-invoice is keyed off **customer country** only (customer in `XX`) | Add `XX` to `eInvoiceCountries`, add `eInvoiceConfig.XX`, extend `EInvoiceConfig` in `configItems.interface.ts`. Often **no** change to `E_INVOICE_PREFIXES` in `DownloadMenu.vue`. |
| E-invoice is keyed off **invoice `businessArea`** (e.g. `VN…`, `EG…`, `BE…`) | Add the **two-letter prefix** to `E_INVOICE_PREFIXES`, add matching `eInvoiceConfig[<prefix>]`, and align `getEInvoiceData()` (ordering and guards such as `eInvoiceNo` for VN). |
| **Both** customer country `XX` **and** business areas `XX…` must work (common for regional billing) | Add `XX` to **`eInvoiceCountries`** (for `showEInvoice`, `getDocumentType` customer branch, and **`getExtension`** when `fileType` is json/xml) **and** add `XX` to **`E_INVOICE_PREFIXES`** with `eInvoiceConfig.XX` so business-area rows resolve. Example: **Belgium (MYF-4368)**. |
| Backend introduces a **new** `DocumentTypeEnum` value | Edit `schemas/myfinance-export-documents-API.v1.yml`, run `npm run codegen-export-documents`, then wire the new enum member in `eInvoiceConfig`. |

**Extension resolution (`getExtension`)**: For `pdfType === "eInvoice"`, the **non-pdf** `fileType` from `eInvoiceConfig` is applied only when the **country code string** passed into `getExtension` is listed in `eInvoiceCountries`. If you add a business-area prefix that uses `xml`/`json`, that prefix must appear in `eInvoiceCountries` so the download uses the right extension. If `fileType` is `pdf`, falling through to `"pdf"` can still work when the prefix is omitted from `eInvoiceCountries` — verify for the new country.

---

## Implementation checklist

Copy and tick through in order.

### 1) Document type: reuse existing enum vs new OpenAPI value

- [ ] Confirm with backend which **`DocumentTypeEnum`** value the export-documents API expects (many rollouts **reuse** an existing member, e.g. **xml** e-invoices using `DocumentTypeEnum.VIETNAMINVOICE`).
- [ ] **Reuse only (no contract change)** — typical “enable a country” ticket: **do not** edit `schemas/` or run `codegen-export-documents`. Import the chosen enum from `@/auto/api/export-documents` and set `eInvoiceConfig[…].invoiceType` to that member.
- [ ] **New enum member**: edit `schemas/myfinance-export-documents-API.v1.yml`, run `npm run codegen-export-documents`, wire the new value into `eInvoiceConfig`, then follow the **openapi-schema-codegen** skill for wrappers, `mock/` handlers, and tests (see [Related skills](#related-skills)).

### 2) Config maps (source of truth for type + extension)

- [ ] `src/lib/configItems.ts`: append the country code to `eInvoiceCountries` (comma-separated; same `XX` used for customer country and/or business-area prefix when both apply).
- [ ] `src/lib/configItems.ts`: add or update `eInvoiceConfig[<key>]` with `{ invoiceType: DocumentTypeEnum.…, fileType: "json" | "xml" | "pdf" }` (plus payload-driven `zip` via `eInvoiceFileType` where applicable).
- [ ] `src/interfaces/configItems.interface.ts`: extend **`EInvoiceConfig`** with the new key(s).

### 3) Download UI and request routing

- [ ] `src/components/DownloadMenu.vue`:
  - [ ] If business-area-driven or **dual** customer+business-area: update **`E_INVOICE_PREFIXES`** (order matters for `getEInvoiceData()`), review **`getEInvoiceData()`** guards.
  - [ ] Treat **`isBusinessArea`** as a **boolean** computed (e.g. whether **any** selected invoice’s `businessArea` matches a prefix). Use it consistently in **`showEInvoice`** and **`getDocumentType("eInvoice")`** — not a filtered list’s `.length` (see MYF-4368 / PR **#4554**).
  - [ ] Confirm **`showEInvoice`** matches product: tabs, customer or business-area gate, and payload flags (`hasJson`, `hasXML`, or e-invoice fields).
  - [ ] Confirm **`getDocumentType("eInvoice")`** matches **`eInvoiceConfig`** for both **business-area substring** and **customer-country** branches.

### 4) Download pipeline (extension, zip prefixes)

- [ ] `src/lib/download.utility.ts`: confirm **`getExtension`** for the new `countryCode` / `eInvoiceFileType`; add unit tests.
- [ ] `src/lib/utilities.ts`: **`getDefaultPrefix`** only special-cases India (`IN`) for e-invoice bulk zip naming — extend only if product requires it.
- [ ] `src/lib/typed-utilities.ts`: **`isDownloadableInvoice`** must stay true for target rows (json, xml, or approved e-invoice per existing rule).

### 5) Invoice payload and local mocks

- [ ] Helpers/stores already map **`hasJson`**, **`hasXML`**, **`hasEInvoice`**, **`eInvoiceStatus`**, **`eInvoiceFileType`**, **`eInvoiceNo`**, **`businessArea`** into `downloadableInvoices` (e.g. open/paid helpers). If not, update mappers.
- [ ] **Vite mock data**: add or adjust a representative invoice in **`mock/data/open-invoices.json`** (and other tab mocks if you test those tabs) with the right **`businessArea`**, flags, and e-invoice fields so local **`/myfinance/api`** behaviour matches the story (see openapi-schema-codegen for mock server layout).

### 6) Tests (match typical PR structure)

- [ ] **`tests/unit/lib/download.utility.spec.ts`**: assert **`getExtension("eInvoice", "<code>", …)`** for the new country/prefix (and `zip` when applicable).
- [ ] **`tests/unit/components/DownloadMenu.spec.ts`**: extend existing cases (tooltips, `documentType`, download wiring) as needed.
- [ ] **`tests/unit/components/DownloadMenu.more.spec.ts`**: use this file (or add it) for **`showEInvoice` matrices** — customer country vs **`businessArea`** vs neither, with **`createTestingPinia()`** and **`useAppStore().customerCountryCode`**. **Set store state after `shallowMount`** when asserting computeds that depend on the app store, so reactive updates match runtime (avoids ordering flakes; see MYF-4368 fix).
- [ ] **Cypress** `tests/component/components/DownloadMenu.spec.ts`: only when you need full component/DOM coverage; not required for every country add.
- [ ] **Collateral Cypress (as needed):** New mock rows or navigation timing can surface flakes in **other** component specs (e.g. `tests/component/components/OpenInvoices.spec.ts` — router baseline, wait for store loading before patching fixtures, stable selectors). Fix in the same PR when CI fails; not part of the minimal e-invoice file set (PR **#4554**).

### 7) Verify

- [ ] `npm run typecheck`
- [ ] `npm run test:unit --` with paths for every touched spec (include **`DownloadMenu.more.spec.ts`** if present)
- [ ] Manual: target customer country and/or business-area invoice on open/paid/credits as applicable; single and multi-select download

---

## Related skills

- **openapi-schema-codegen** — In the ui-myfinance repo, use `.cursor/skills/openapi-schema-codegen/SKILL.md`, or the mirrored copy under `myf-agent-skills/openapi-schema-codegen/SKILL.md` when your checkout includes it. Covers `schemas/`, `npm run codegen-*`, `src/auto/api/`, mocks, and verification when the export-documents contract changes.
- **vitest-unit-component-testing** — In a full ui-myfinance checkout with agent skills, see `myf-agent-skills/vitest-unit-component-testing/SKILL.md` for Vitest + Pinia patterns.

---

## Quick reference (key symbols)

| Symbol | Role |
|--------|------|
| `configItems.eInvoiceCountries` | Comma-separated list; drives `showEInvoice` and `getExtension` eligibility |
| `configItems.eInvoiceConfig` | Per-key `invoiceType` + `fileType` |
| `E_INVOICE_PREFIXES` | Business-area prefixes; `getEInvoiceData()` + `countryCode` passed to download |
| `isBusinessArea` | `DownloadMenu.vue` computed **boolean**; true when any row’s `businessArea` matches a prefix |
| `DocumentTypeEnum` | Generated from export-documents OpenAPI |

Deeper behaviour and **MYF-4368 step order**: [references/flow-and-files.md](references/flow-and-files.md).
