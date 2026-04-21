# E-invoice flow: files and behaviour

This reference expands [SKILL.md](../SKILL.md) with concrete file roles, test patterns, and a reference checklist.

## End-to-end flow

1. User opens a tab that lists invoices (`openInvoices`, `overdueInvoices`, `paidInvoices`, `creditedInvoices`).
2. Row or bulk selection builds `downloadableInvoices` with payload flags.
3. `DownloadMenu.vue` decides whether the **e-invoice** option appears (`showEInvoice`).
4. On click, `downloadPdf("eInvoice")` resolves `documentType`, `pdfTemplate`, and a **country code** passed to `downloadMultipleFiles({ countryCode })`.
5. `download.utility.ts` loads files, uses `getExtension(pdfType, countryCode, eInvoiceFileType)` to set the file extension and MIME handling, and may zip multiple results.

## Primary files

### `src/lib/configItems.ts`

- **`eInvoiceCountries`**: Comma-separated string. Used for:
  - Gating **visibility** of e-invoice in the menu (together with business-area logic).
  - Resolving **non-pdf** extensions in `getExtension()` — the `countryCode` argument must match an entry here when `fileType` is `json` or `xml`.
- **`eInvoiceConfig`**: Keys are **either** ISO country codes **or** business-area prefixes (two letters). Each value has:
  - **`invoiceType`**: A member of generated `DocumentTypeEnum` (often **reused** across countries, e.g. `VIETNAMINVOICE` for xml in more than one jurisdiction).
  - **`fileType`**: `"json" | "xml" | "pdf"`; **`zip`** via invoice payload `eInvoiceFileType` in `getExtension`.

### `src/interfaces/configItems.interface.ts`

- **`EInvoiceConfig`**: Extend with every new key in `eInvoiceConfig`.

### `src/components/DownloadMenu.vue`

- **`E_INVOICE_PREFIXES`**: Ordered list of two-letter **business-area** prefixes. `getEInvoiceData()` returns the **first** prefix that has matching rows (VN requires `eInvoiceNo`).
- **`isBusinessArea`**: A **`computed<boolean>`** — true if **any** selected invoice’s `businessArea` starts with one of those prefixes (typically via `.some()` over invoices and prefixes). **`showEInvoice`** and **`getDocumentType("eInvoice")`** branch on this boolean, not on a filtered array’s length.
- **`showEInvoice`**: Tab allowlist AND (`eInvoiceCountries` contains customer country OR `isBusinessArea`) AND (json OR xml OR e-invoice flags).
- **`getDocumentType("eInvoice")`**: Business-area branch uses `eInvoiceConfig[businessArea.substring(0, 2)]`; customer branch uses `eInvoiceConfig[customerCountryCode]` when country is in `eInvoiceCountries`.
- **`downloadPdf` → `downloadMultipleFiles`**: `countryCode` is `eInvoiceData.code` (prefix) when business-area data wins, else **`customerCountryCode`**.

### `src/lib/download.utility.ts`

- **`getExtension`**: Non-pdf `fileType` from `eInvoiceConfig` applies when `countryCode` is in **`eInvoiceCountries`**. Prefixes using **xml/json** must appear in `eInvoiceCountries` for correct extensions.

### `src/lib/utilities.ts`

- **`getDefaultPrefix`**: Only **`IN`** uses the India-specific chunk prefix for e-invoice bulk zips unless extended.
- **Typical country-enable work:** **read and confirm** behaviour only; **do not change** this file unless product requires a new bulk-zip prefix rule. Enabling e-invoice for a new country does not usually require edits here.

### `src/lib/typed-utilities.ts`

- **`isDownloadableInvoice`**: For `eInvoice`, requires `hasJson || hasXML || (hasEInvoice && eInvoiceStatus === "APPROVED")`.
- **Typical country-enable work:** **read and confirm** rows for the new country still satisfy this; **do not change** unless eligibility rules differ. Enabling e-invoice for a new country does not usually require edits here.

### Invoice mappers and stores

- Pass through **`hasJson`**, **`hasXML`**, **`hasEInvoice`**, **`eInvoiceStatus`**, **`eInvoiceFileType`**, **`eInvoiceNo`**, **`businessArea`** into rows used by `DownloadMenu`.

### Local dev mocks (`mock/data/`)

- **`mock/data/open-invoices.json`**: add a row with the target **`businessArea`**, **`hasXML`** / **`hasJson`** / **`hasEInvoice`**, **`eInvoiceStatus`**, etc., so the Vite mock server (`/myfinance/api`) exercises the same paths as production data for the **open** tab. When routes or bodies change, align mock expectations with the **openapi-schema-codegen** companion skill (see [SKILL.md](../SKILL.md) → Related skills).
- **Other tabs:** when acceptance criteria or automated tests cover **paid**, **credited**, overdue, or disputed flows, add matching sample rows to the corresponding files (e.g. **`mock/data/paid-invoices.json`**, **`mock/data/credited-invoices.json`**, and other `mock/data/*` payloads those views load). A minimal **open-tab-only** change often updates **`open-invoices.json`** alone; multi-tab stories should update every mock the UI hits.

## OpenAPI and codegen (only when adding a new document type)

- **Schema**: `schemas/myfinance-export-documents-API.v1.yml`.
- **Command**: `npm run codegen-export-documents` → `src/auto/api/export-documents.ts`.
- Many country tickets **reuse** an existing `DocumentTypeEnum` value — then **no** schema edit and **no** codegen run.

## Tests (Vitest and optional Cypress)

| File | Role |
|------|------|
| `tests/unit/lib/download.utility.spec.ts` | **`getExtension`** cases for new country code / prefix |
| `tests/unit/components/DownloadMenu.spec.ts` | Existing DownloadMenu behaviour, tooltips, `documentType` |
| `tests/unit/components/DownloadMenu.more.spec.ts` | Large **`showEInvoice`** matrices (customer vs `businessArea`); use **`createTestingPinia()`**; set **`useAppStore().customerCountryCode`** after **`shallowMount`** when testing computeds that depend on the store (same ordering as in that spec file) |
| `tests/component/components/DownloadMenu.spec.ts` | Optional Cypress component coverage |

### Collateral Cypress (stability)

Adding or changing mock invoices can affect **other** component tests (routing, async load, tag clicks). For example, **`OpenInvoices.spec.ts`** sometimes needs a stable router baseline, waiting until `isLoading` is false before patching store data, and robust selectors. Apply similar fixes when CI fails, even if the file is outside the core checklist above.

## Pinia / app store in unit tests

When the SFC reads **`useAppStore().customerCountryCode`** inside computeds like `showEInvoice`, mutate the store **after** the component is mounted with **`createTestingPinia()`** if you see stale or flaky assertions — mirror the working pattern in **`DownloadMenu.more.spec.ts`**.

## Reference checklist: Belgium (BE)

Following the skill in order should yield the same **shape** of change for a **dual** customer-country + business-area country (example **Belgium**):

1. **`configItems.interface.ts`**: add **`BE`** to **`EInvoiceConfig`**.
2. **`configItems.ts`**: append **`BE`** to **`eInvoiceCountries`**; add **`eInvoiceConfig.BE`** with **`invoiceType: DocumentTypeEnum.VIETNAMINVOICE`**, **`fileType: "xml"`** (reuse enum — no codegen).
3. **`DownloadMenu.vue`**: add **`BE`** to **`E_INVOICE_PREFIXES`**; keep **`getEInvoiceData()`** / **`isBusinessArea`** consistent with product rules.
4. **`download.utility.spec.ts`**: **`getExtension`** for **`BE`**.
5. **`DownloadMenu.spec.ts`** / **`DownloadMenu.more.spec.ts`**: **`showEInvoice`** true when `customerCountryCode === "BE"` or `businessArea` starts with **`BE`**, false otherwise; respect Pinia ordering above.
6. **`mock/data/open-invoices.json`**: sample **BE** invoice with **`hasXML`**, **`hasEInvoice`**, **`eInvoiceStatus`**, **`businessArea`** like **`BE00`**.
7. **Optional (when CI requires it):** stabilize unrelated Cypress specs that mock data or navigation touched — see **Collateral Cypress** above (e.g. **`OpenInvoices.spec.ts`**).

## Verification commands

```bash
npm run typecheck
npm run test:unit -- tests/unit/components/DownloadMenu.spec.ts tests/unit/components/DownloadMenu.more.spec.ts tests/unit/lib/download.utility.spec.ts
```

Omit `DownloadMenu.more.spec.ts` if that file was not added or touched.
