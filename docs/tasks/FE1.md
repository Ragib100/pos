# FE1 · Terminal (cashier-facing)

Maintains: `frontend/src/app/(terminal)/**` and terminal-only components. Anyone else changes this code through a pull request that you review. In other lanes' code, change only what a task in your file needs, and have its maintainer review it.
Start a task: open Claude Code in the repo folder and type **`I am FE1. Do F1-00.`** (replace `FE1` with your name once the team is assigned). Total: 15 tasks, one per milestone. `[ ]` = todo, `[~]` = in progress, `[x]` = merged to main.
Sizes are relative (S, M, L) and will be re-estimated after M0. Support tasks are one step smaller than their milestone.
Frontend tasks build against **mocks generated from the OpenAPI schema** until the matching backend task is merged. Backend tasks publish the schema for their endpoints before building them (`CONTRACTS.md`).

---

### F1-00 · App shell, proxy, API client and shortcuts `[ ]`

Size M · Milestone M0 (Foundations) · Depends: none · Unblocks: M1, M2 · Gate: G1 (frontend tasks)

- Next.js app shell.
- Proxy to Django (no CORS).
- API client generation from the OpenAPI schema.
- Data-fetching and error-handling patterns.
- Keyboard shortcut framework.
- Local draft storage utility.
- **API operations:** section 4.16 of `API.md`.
- **Done when:** your part of the M0 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-01 · Terminal login, PIN login and context selection `[ ]`

Size L · Milestone M1 (Identity, tenancy and admin) · Depends: M0 · Unblocks: M4 · Gate: G2

- Terminal login and PIN login screens.
- Shop, branch and register selection.
- Screen lock after inactivity.
- Session expiry handling.
- **Contract from:** B1-01, B2-01. Use the mock until the real endpoints are merged.
- **Pages:** PG-02 to PG-05 in `PAGES.md`.
- **API operations:** section 4.1 to 4.4 of `API.md`.
- **Done when:** your part of the M1 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-02 · POS product lookup `[ ]`

Size L · Milestone M2 (Catalog and pricing) · Depends: M0 · Unblocks: M3

- Product lookup component for the POS (scan or search by name or SKU, variant picker).
- Keyboard-first behaviour.
- **Contract from:** B1-02, B2-02. Use the mock until the real endpoints are merged.
- **API operations:** section 4.5 of `API.md`.
- **Pages:** the search and scan box of PG-40 (and the product picker, section 19 of `PAGES.md`).
- **Done when:** your part of the M2 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-03 · Terminal stock lookup `[ ]`

Size M · Milestone M3 (Inventory) · Depends: M2 · Unblocks: M5 · Gate: G3

- Quick stock lookup at the terminal (is it in stock, which batch expires first).
- **Contract from:** B1-03, B2-03. Use the mock until the real endpoints are merged.
- **API operations:** section 4.6 of `API.md`.
- **Pages:** the stock and batch hint of PG-40 in `PAGES.md`.
- **Done when:** your part of the M3 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-04 · Open-shift, cash and Z-report screens `[ ]`

Size M · Milestone M4 (Shifts and cash) · Depends: M1 · Unblocks: M5

- Open-shift screen with opening cash.
- Cash in and cash out with reasons.
- Close-shift and cash count.
- Z-report view and print layout.
- **Contract from:** B1-04, B2-04. Use the mock until the real endpoints are merged.
- **Pages:** PG-34 to PG-36 in `PAGES.md`.
- **API operations:** section 4.2 and 4.7 of `API.md`.
- **Done when:** your part of the M4 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-05 · POS screen and payments `[ ]`

Size L · Milestone M5 (Billing and payments) · Depends: M3, M4 · Unblocks: M6, M7, M9 · Gate: G4

- The POS screen: cart, scan or search, quantity and weight entry, discounts, hold and recall, payment dialog (cash with change, card and MFS with transaction ID, split payments), void with approval.
- **full keyboard operation**.
- Draft bill survives refresh and power cut.
- **Contract from:** B1-05, B2-05. Use the mock until the real endpoints are merged.
- **Pages:** PG-40 to PG-41 in `PAGES.md`.
- **API operations:** section 4.8 of `API.md`.
- **Done when:** your part of the M5 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-06 · Receipt printing and hardware `[ ]`

Size M · Milestone M6 (Receipts and hardware) · Depends: M5 · Unblocks: M7 · Gate: G5

- Receipt printing on 58 mm and 80 mm thermal printers (Bangla and English).
- Cash drawer opening.
- Barcode scanner handling.
- Optional scale and label printer.
- Duplicate-copy reprints.
- **First:** a short test on the real printer (print, cut, drawer kick) through the local print service or WebUSB. If it fails, raise it with the team (D-027).
- Installable web app delivery for desktop and tablet, plus the small local print service (D-027).
- **Contract from:** B1-06, B2-06. Use the mock until the real endpoints are merged.
- **Pages:** PG-40, PG-43 in `PAGES.md`.
- **API operations:** section 4.9 of `API.md`.
- **Done when:** your part of the M6 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-07 · End-of-shift summary `[ ]`

Size S · Milestone M7 (Reports v1) · Depends: M5, M6 · Unblocks: M8, M10, M11

- Cashier end-of-shift summary.
- **Contract from:** B1-07, B2-07. Use the mock until the real endpoints are merged.
- **Pages:** PG-42 in `PAGES.md`.
- **API operations:** section 4.10 of `API.md`.
- **Done when:** your part of the M7 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-08 · Customer attach, credit and loyalty at the counter `[ ]`

Size L · Milestone M8 (Customers, credit and loyalty) · Depends: M7 · Unblocks: M9, M12

- Attach customer at the POS by mobile number.
- Credit payment.
- Loyalty and gift card redemption.
- Collect a due at the counter.
- **Contract from:** B1-08, B2-08. Use the mock until the real endpoints are merged.
- **Pages:** PG-54 in `PAGES.md`.
- **API operations:** section 4.11 of `API.md`.
- **Done when:** your part of the M8 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-09 · Returns at the counter `[ ]`

Size M · Milestone M9 (Returns and exchanges) · Depends: M5, M8 · Unblocks: M12

- Find bill, select lines to return, choose refund type, exchange flow, print refund slip.
- **Contract from:** B1-09, B2-09. Use the mock until the real endpoints are merged.
- **Pages:** PG-55 in `PAGES.md`.
- **API operations:** section 4.12 of `API.md`.
- **Done when:** your part of the M9 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-10 · Goods receipt at the terminal `[ ]`

Size M · Milestone M10 (Suppliers and purchasing) · Depends: M7 · Unblocks: M12

- Quick goods receipt at the terminal if needed (stock keeper flow).
- **Contract from:** B1-10, B2-10. Use the mock until the real endpoints are merged.
- **Pages:** PG-63 in `PAGES.md`.
- **API operations:** section 4.13 of `API.md`.
- **Done when:** your part of the M10 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-11 · Offers and coupons at the counter `[ ]`

Size M · Milestone M11 (Promotions and coupons) · Depends: M7 · Unblocks: M12

- Offers applied automatically at the POS with a clear breakdown.
- Coupon entry.
- **Contract from:** B1-11, B2-11. Use the mock until the real endpoints are merged.
- **API operations:** section 4.14 of `API.md`.
- **Pages:** the offers banner and coupon box of PG-40 in `PAGES.md`.
- **Done when:** your part of the M11 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-12 · In-app alerts and expense entry `[ ]`

Size L · Milestone M12 (Expenses, reports and alerts) · Depends: M8, M9, M10, M11 · Unblocks: M13

- In-app alerts at the terminal.
- Expense entry from the drawer.
- **Contract from:** B1-12, B2-12. Use the mock until the real endpoints are merged.
- **Pages:** PG-66, PG-68 in `PAGES.md`.
- **API operations:** section 4.15 of `API.md`.
- **Done when:** your part of the M12 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-13 · Preset behaviour at the counter `[ ]`

Size M · Milestone M13 (Shop-type presets) · Depends: M12 · Unblocks: M14

- Preset-specific counter behaviour: weighed items (grocery), batch and expiry pick (pharmacy), variant picker (clothing).
- **Contract from:** B1-13, B2-13. Use the mock until the real endpoints are merged.
- **Pages:** the module-dependent parts of PG-40 (see section 18 of `PAGES.md`): weight prompt, batch choice, variant picker.
- **Done when:** your part of the M13 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F1-14 · Counter performance and hardware QA `[ ]`

Size L · Milestone M14 (Hardening and launch readiness) · Depends: M13 · Unblocks: none · Gate: G6

- Counter performance (scan within 1 second, bill within 3 seconds).
- Hardware QA on real devices.
- Draft-bill and power-cut tests.
- Touch and keyboard usability.
- **Contract from:** B1-14, B2-14. Use the mock until the real endpoints are merged.
- **API operations:** section 4.16 of `API.md`.
- **Pages:** PG-40, PG-41 and PG-43 (performance and hardware checks).
- **Done when:** your part of the M14 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

---

## Log (newest last: date · task · what changed · notes for next task)

_No entries yet._