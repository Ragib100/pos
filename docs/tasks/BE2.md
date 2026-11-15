# BE2 · Catalog, stock and back office

Owns: `backend/apps/catalog/**`, `backend/apps/inventory/**`, `backend/apps/customers/**`, `backend/apps/purchasing/**`, `backend/apps/reports/**`, `scripts/**`. Touch another lane's code only for a task listed in your file, and have that lane's owner review it.
Start a task: open Claude Code in the repo folder and type **`I am BE2. Do B2-00.`** (replace `BE2` with your name once the team is assigned). Total: 15 tasks, one per milestone. `[ ]` = todo, `[~]` = in progress, `[x]` = merged to main.
Sizes are relative (S, M, L) and will be re-estimated after M0. Support tasks are one step smaller than their milestone.
Frontend tasks build against **mocks generated from the OpenAPI schema** until the matching backend task is merged. Backend tasks publish the schema for their endpoints before building them (`CONTRACTS.md`).

---

### B2-00 · RLS migration helpers, CI checks, OpenAPI and fixtures `[ ]`

Size M · Milestone M0 (Foundations) · Depends: none · Unblocks: M1, M2 · Gate: G1 (frontend tasks)

- Migration helpers for RLS policies and `shop_id`-first indexes.
- CI checks (RLS coverage test, import-boundary test, schema-up-to-date check).
- Drf-spectacular setup and committed schema.
- Test fixtures with several shops.
- Seed command skeleton.
- Settings for Bangla and time zone.
- **API operations:** section 4.16 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M0 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-01 · Shop-type presets, settings and default roles `[ ]`

Size L · Milestone M1 (Identity, tenancy and admin) · Depends: M0 · Unblocks: M4 · Gate: G2

- `shop_types` presets and seed data (generic, grocery, pharmacy, clothing: modules, attributes, units, tax defaults).
- `shops.enabled_modules` and settings APIs.
- Permission catalog and default roles per shop.
- Shop details (name, logo, BIN, phone, email).
- **API operations:** section 4.1 to 4.4 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M1 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-02 · Catalog models, search, import and export `[ ]`

Size L · Milestone M2 (Catalog and pricing) · Depends: M0 · Unblocks: M3

- `categories`, `brands`, `units`, `tax_rates`, `attribute_definitions`, `products`, `product_variants`, `variant_barcodes`, `unit_conversions`, `combo_items`, `branch_variants`.
- Custom attributes in JSONB.
- CSV and Excel import and export.
- Search over 50,000 products with Bangla support.
- Barcode generation.
- **API operations:** section 4.5 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M2 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-03 · Inventory ledger, batches and stock reports `[ ]`

Size L · Milestone M3 (Inventory) · Depends: M2 · Unblocks: M5 · Gate: G3

- `stock_batches`, `stock_levels`, `stock_movements`, `stock_adjustment_reasons`, `stock_adjustments` and items, `stock_counts` and items.
- Opening stock with batch and expiry.
- Negative-stock policy.
- First-expiring-first-out selection.
- Weighted average cost.
- Low-stock and expiry queries.
- Stock report and stock value.
- **API operations:** section 4.6 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M3 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-04 · Number sequences and shift constraint review `[ ]`

Size S · Milestone M4 (Shifts and cash) · Depends: M1 · Unblocks: M5

- `number_sequences` (per register or branch).
- Review of shift constraints and tests.
- Shift reports query base.
- **API operations:** section 4.2 and 4.7 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M4 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-05 · Stock deduction and price resolution in sales `[ ]`

Size L · Milestone M5 (Billing and payments) · Depends: M3, M4 · Unblocks: M6, M7, M9 · Gate: G4

- Stock deduction through the stock service (batches, weighted items, combos, unit conversions).
- Price resolution in the sale flow.
- Sale-related queries.
- Performance tests (bill completed within 3 seconds).
- **API operations:** section 4.8 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M5 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-06 · Label data and receipt template settings `[ ]`

Size M · Milestone M6 (Receipts and hardware) · Depends: M5 · Unblocks: M7 · Gate: G5

- Label data for barcode printing.
- Receipt template settings API.
- **API operations:** section 4.9 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M6 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-07 · Sales reports and reconciliation checks `[ ]`

Size S · Milestone M7 (Reports v1) · Depends: M5, M6 · Unblocks: M8, M10, M11

- Sales reports by day, week, month and custom range.
- Stock report integration.
- Reconciliation checks.
- **API operations:** section 4.10 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M7 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-08 · Customers, ledgers, loyalty and gift cards `[ ]`

Size L · Milestone M8 (Customers, credit and loyalty) · Depends: M7 · Unblocks: M9, M12

- `customer_groups`, `group_prices`, `customers`, `customer_ledger`, `due_collections`, `loyalty_transactions`, `gift_cards`, `gift_card_transactions`.
- Due balances from the ledger.
- Loyalty earn and redeem rules.
- Overdue due list.
- **API operations:** section 4.11 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M8 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-09 · Return stock effects `[ ]`

Size M · Milestone M9 (Returns and exchanges) · Depends: M5, M8 · Unblocks: M12

- Stock effects: good items back to sellable stock, damaged items to a loss adjustment.
- Batch handling on returns.
- **API operations:** section 4.12 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M9 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-10 · Suppliers, orders, receipts and purchase ledger `[ ]`

Size L · Milestone M10 (Suppliers and purchasing) · Depends: M7 · Unblocks: M12

- `suppliers`, `purchase_orders` and items, `goods_receipts` and items, `purchase_returns` and items, `supplier_payments`, `supplier_ledger`.
- Partial receipts.
- Unit received.
- Batch and expiry on receipt.
- Average cost update.
- Purchase discount and VAT.
- **API operations:** section 4.13 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M10 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-11 · Promotion review and reporting data `[ ]`

Size S · Milestone M11 (Promotions and coupons) · Depends: M7 · Unblocks: M12

- Review of the engine against stock and pricing.
- Promotion reports data.
- **API operations:** section 4.14 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M11 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-12 · Reports, VAT report and exports `[ ]`

Size L · Milestone M12 (Expenses, reports and alerts) · Depends: M8, M9, M10, M11 · Unblocks: M13

- Reports: by product, category, cashier, branch and payment method.
- Best and worst sellers.
- Profit and loss.
- Customer and supplier dues.
- VAT report (sales VAT and purchase VAT).
- PDF and Excel export.
- **API operations:** section 4.15 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M12 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-13 · Preset definitions, seed data and sample catalogs `[ ]`

Size M · Milestone M13 (Shop-type presets) · Depends: M12 · Unblocks: M14

- Final preset definitions and seed data per type (modules, custom fields, units, tax, receipt defaults).
- Sample catalogs for demos.
- **Done when:** your part of the M13 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B2-14 · Backups, export, import tools and runbooks `[ ]`

Size L · Milestone M14 (Hardening and launch readiness) · Depends: M13 · Unblocks: none · Gate: G6

- Daily backups with 30-day retention and restore drills.
- Per-shop data export.
- Data import tools for existing shop data.
- Migration runbooks.
- **API operations:** section 4.16 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M14 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

---

## Log (newest last: date · task · what changed · notes for next task)

_No entries yet._