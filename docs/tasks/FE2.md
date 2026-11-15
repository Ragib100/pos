# FE2 · Admin and back office

Owns: `frontend/src/app/(admin)/**`, the shared design system and shared components. Touch another lane's code only for a task listed in your file, and have that lane's owner review it.
Start a task: open Claude Code in the repo folder and type **`I am FE2. Do F2-00.`** (replace `FE2` with your name once the team is assigned). Total: 15 tasks, one per milestone. `[ ]` = todo, `[~]` = in progress, `[x]` = merged to main.
Sizes are relative (S, M, L) and will be re-estimated after M0. Support tasks are one step smaller than their milestone.
Frontend tasks build against **mocks generated from the OpenAPI schema** until the matching backend task is merged. Backend tasks publish the schema for their endpoints before building them (`CONTRACTS.md`).

---

### F2-00 · Design system, i18n and form building blocks `[ ]`

Size M · Milestone M0 (Foundations) · Depends: none · Unblocks: M1, M2 · Gate: G1 (frontend tasks)

- Design tokens and base component library (touch-friendly, large buttons).
- Bangla and English i18n with Bangla fonts.
- Layout and navigation.
- Table, form and dynamic-form building blocks.
- Mock API server from the schema.
- **API operations:** section 4.16 of `API.md`.
- **Done when:** your part of the M0 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-01 · Login, platform admin and user management `[ ]`

Size L · Milestone M1 (Identity, tenancy and admin) · Depends: M0 · Unblocks: M4 · Gate: G2

- Login screen.
- Platform-admin panel (create shop from preset, suspend).
- User, role and permission management.
- Branch and register management.
- Shop settings and language switch (Bangla and English digits).
- **Contract from:** B1-01, B2-01. Use the mock until the real endpoints are merged.
- **Pages:** PG-01, PG-06 to PG-16 in `PAGES.md`.
- **API operations:** section 4.1 to 4.4 of `API.md`.
- **Done when:** your part of the M1 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-02 · Catalog screens and import wizard `[ ]`

Size L · Milestone M2 (Catalog and pricing) · Depends: M0 · Unblocks: M3

- Category, brand, unit and tax screens.
- Product list and search.
- Product form generated from `attribute_definitions`.
- Variant editor.
- Barcode labels.
- Import and export wizard.
- **Contract from:** B1-02, B2-02. Use the mock until the real endpoints are merged.
- **Pages:** PG-17 to PG-24 in `PAGES.md`.
- **API operations:** section 4.5 of `API.md`.
- **Done when:** your part of the M2 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-03 · Inventory screens `[ ]`

Size L · Milestone M3 (Inventory) · Depends: M2 · Unblocks: M5 · Gate: G3

- Opening stock entry.
- Stock levels and movement history.
- Adjustments with reasons and approval.
- Stock count screens.
- Low-stock and expiry lists.
- Stock report.
- **Contract from:** B1-03, B2-03. Use the mock until the real endpoints are merged.
- **Pages:** PG-26 to PG-31 in `PAGES.md`.
- **API operations:** section 4.6 of `API.md`.
- **Done when:** your part of the M3 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-04 · Shift history and approval dialog `[ ]`

Size M · Milestone M4 (Shifts and cash) · Depends: M1 · Unblocks: M5

- Shift history and Z-report lookup for managers and owners.
- Manager approval dialog (PIN on the cashier's screen) shared component.
- **Contract from:** B1-04, B2-04. Use the mock until the real endpoints are merged.
- **Pages:** PG-37 to PG-39 in `PAGES.md`.
- **API operations:** section 4.2 and 4.7 of `API.md`.
- **Done when:** your part of the M4 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-05 · Payment settings and sales list `[ ]`

Size L · Milestone M5 (Billing and payments) · Depends: M3, M4 · Unblocks: M6, M7, M9 · Gate: G4

- Payment method settings.
- Sales list and bill lookup for managers.
- Tax and rounding settings.
- Support FE1 on shared components.
- **Contract from:** B1-05, B2-05. Use the mock until the real endpoints are merged.
- **Pages:** PG-47 to PG-48 in `PAGES.md`.
- **API operations:** section 4.8 of `API.md`.
- **Done when:** your part of the M5 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-06 · Receipt template editor and label designer `[ ]`

Size M · Milestone M6 (Receipts and hardware) · Depends: M5 · Unblocks: M7 · Gate: G5

- Receipt template editor (logo, header, footer, language, QR).
- Label designer and print.
- Hardware settings screen.
- **Contract from:** B1-06, B2-06. Use the mock until the real endpoints are merged.
- **Pages:** PG-25, PG-43 to PG-44 in `PAGES.md`.
- **API operations:** section 4.9 of `API.md`.
- **Done when:** your part of the M6 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-07 · Dashboard and sales reports `[ ]`

Size S · Milestone M7 (Reports v1) · Depends: M5, M6 · Unblocks: M8, M10, M11

- Dashboard (sales, bills, average bill, profit, low stock).
- Sales reports by period.
- Report filters.
- **Contract from:** B1-07, B2-07. Use the mock until the real endpoints are merged.
- **Pages:** PG-33, PG-45 to PG-46 in `PAGES.md`.
- **API operations:** section 4.10 of `API.md`.
- **Done when:** your part of the M7 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-08 · Customer and gift card screens `[ ]`

Size L · Milestone M8 (Customers, credit and loyalty) · Depends: M7 · Unblocks: M9, M12

- Customer list and profile (history, dues, points).
- Customer groups and group prices.
- Gift card management.
- Overdue dues screen.
- **Contract from:** B1-08, B2-08. Use the mock until the real endpoints are merged.
- **Pages:** PG-49 to PG-53 in `PAGES.md`.
- **API operations:** section 4.11 of `API.md`.
- **Done when:** your part of the M8 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-09 · Return history and settings `[ ]`

Size M · Milestone M9 (Returns and exchanges) · Depends: M5, M8 · Unblocks: M12

- Return history and reports.
- Return-window setting.
- **Contract from:** B1-09, B2-09. Use the mock until the real endpoints are merged.
- **Pages:** PG-56 in `PAGES.md`.
- **API operations:** section 4.12 of `API.md`.
- **Done when:** your part of the M9 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-10 · Supplier and purchasing screens `[ ]`

Size L · Milestone M10 (Suppliers and purchasing) · Depends: M7 · Unblocks: M12

- Supplier screens.
- Purchase order create and print.
- Goods receipt (partial, batch, expiry, unit).
- Purchase returns.
- Supplier payments and balance.
- **Contract from:** B1-10, B2-10. Use the mock until the real endpoints are merged.
- **Pages:** PG-57 to PG-62 in `PAGES.md`.
- **API operations:** section 4.13 of `API.md`.
- **Done when:** your part of the M10 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-11 · Promotion and coupon screens `[ ]`

Size M · Milestone M11 (Promotions and coupons) · Depends: M7 · Unblocks: M12

- Promotion and coupon management screens.
- Schedule and priority settings.
- **Contract from:** B1-11, B2-11. Use the mock until the real endpoints are merged.
- **Pages:** PG-64 to PG-65 in `PAGES.md`.
- **API operations:** section 4.14 of `API.md`.
- **Done when:** your part of the M11 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-12 · Notification centre, expenses and reports `[ ]`

Size L · Milestone M12 (Expenses, reports and alerts) · Depends: M8, M9, M10, M11 · Unblocks: M13

- Notification centre (read and unread).
- Expense screens.
- All report screens, filters, export and print.
- **Contract from:** B1-12, B2-12. Use the mock until the real endpoints are merged.
- **Pages:** PG-66 to PG-67, PG-69 to PG-74 in `PAGES.md`.
- **API operations:** section 4.15 of `API.md`.
- **Done when:** your part of the M12 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-13 · Preset-specific admin forms `[ ]`

Size M · Milestone M13 (Shop-type presets) · Depends: M12 · Unblocks: M14

- Preset-specific admin forms and views: pharmacy fields and expiry views, clothing size and colour matrix, grocery unit conversion screens.
- **Contract from:** B1-13, B2-13. Use the mock until the real endpoints are merged.
- **Pages:** PG-12, PG-21, PG-23, PG-32 in `PAGES.md`.
- **Done when:** your part of the M13 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### F2-14 · Bangla QA, usability and accessibility `[ ]`

Size L · Milestone M14 (Hardening and launch readiness) · Depends: M13 · Unblocks: none · Gate: G6

- Bangla QA across screens, receipts and reports.
- Usability pass for a 30-minute cashier training.
- Accessibility and error states.
- Help and onboarding text.
- **Contract from:** B1-14, B2-14. Use the mock until the real endpoints are merged.
- **Pages:** PG-75 in `PAGES.md`.
- **API operations:** section 4.16 of `API.md`.
- **Done when:** your part of the M14 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

---

## Log (newest last: date · task · what changed · notes for next task)

_No entries yet._