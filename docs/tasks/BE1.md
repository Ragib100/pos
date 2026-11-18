# BE1 · Core and transactions

Maintains: `backend/apps/accounts/**`, `backend/apps/sales/**`, `backend/apps/promotions/**`, `backend/apps/audit/**`, project settings, CI and `infra/**`. Anyone else changes this code through a pull request that you review. In other lanes' code, change only what a task in your file needs, and have its maintainer review it.
Start a task: open Claude Code in the repo folder and type **`I am BE1. Do B1-00.`** (replace `BE1` with your name once the team is assigned). Total: 15 tasks, one per milestone. `[ ]` = todo, `[~]` = in progress, `[x]` = merged to main.
Sizes are relative (S, M, L) and will be re-estimated after M0. Support tasks are one step smaller than their milestone.
Frontend tasks build against **mocks generated from the OpenAPI schema** until the matching backend task is merged. Backend tasks publish the schema for their endpoints before building them (`CONTRACTS.md`).

---

### B1-00 · Django skeleton, DB roles, tenant context, CI `[ ]`

Size M · Milestone M0 (Foundations) · Depends: none · Unblocks: M1, M2 · Gate: G1 (frontend tasks)

- Django project and app skeleton (`accounts`, `catalog`, `inventory`, `sales`, `customers`, `purchasing`, `promotions`, `audit`, and a read-only `reports`).
- PostgreSQL with separate migration and app roles.
- **tenant context first (D-004)**: per-request `set_config` in a transaction.
- Custom user model.
- Shared abstract tenant base model.
- UUIDv7 keys.
- CI pipeline.
- Docker dev environment.
- Create the GitHub Project, milestones M0 to M14, lane labels and one issue per task ID (a script can read `docs/tasks/`), and the CI workflows with path filters (D-022).
- Create `contracts/` with the OpenAPI schema generation and the generated TypeScript types.
- **API operations:** section 4.16 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M0 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-01 · Auth, sessions, tenancy models and platform admin `[ ]`

Size L · Milestone M1 (Identity, tenancy and admin) · Depends: M0 · Unblocks: M4 · Gate: G2

- `users`, `shops`, `shop_types`, `branches`, `registers`, `roles`, `permissions`, `role_permissions`, `memberships`, `membership_branches`, `auth_sessions`.
- Session auth (D-020): login, select context, token rotation, logout, revoke, "log out everywhere".
- PIN login on registered devices.
- Lockout and throttling.
- At-least-one-owner rule.
- `audit_logs` and `approvals`.
- Platform-admin APIs (create, suspend shop, assign first owner).
- Session lifetimes by client type (terminal about one working day, mobile longer) as server settings, and the shop setting `idle_lock_minutes` (D-024).
- **API operations:** section 4.1 to 4.4 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M1 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-02 · Pricing resolution service and catalog review `[ ]`

Size M · Milestone M2 (Catalog and pricing) · Depends: M0 · Unblocks: M3

- Support BE2: pricing resolution service (group price, then branch price, then default).
- Review of catalog migrations and RLS.
- **API operations:** section 4.5 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M2 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-03 · Stock-posting service and ledger protection `[ ]`

Size M · Milestone M3 (Inventory) · Depends: M2 · Unblocks: M5 · Gate: G3

- Review stock concurrency (row locks, transactions).
- The stock-posting service the sales module will call.
- Protection that blocks updates and deletes on ledger tables in the database.
- **API operations:** section 4.6 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M3 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-04 · Shifts, cash movements and Z-report `[ ]`

Size M · Milestone M4 (Shifts and cash) · Depends: M1 · Unblocks: M5

- `shifts`, `cash_movements`, `z_reports`.
- One open shift per user and per register (partial unique indexes).
- Open and close rules.
- Expected-cash calculation.
- Frozen Z-report snapshot.
- Manager approval flow.
- **API operations:** section 4.2 and 4.7 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M4 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-05 · Sales, payments and the sale transaction `[ ]`

Size L · Milestone M5 (Billing and payments) · Depends: M3, M4 · Unblocks: M6, M7, M9 · Gate: G4

- `sales`, `sale_items`, `payment_methods`, `payments`.
- One transaction per sale: bill, lines, payments, stock movements, shift totals.
- `client_request_id` idempotency.
- Bill numbers.
- VAT, service charge and rounding rules.
- Manual item and bill discounts with role limits.
- Hold and recall.
- Void while the shift is open with reversing entries.
- Snapshot lines.
- Bill numbers use the branch and register prefix, continuous, no reset (D-026).
- VAT is rounded once per bill: add the VAT of all lines (each keeps two decimals), then round half up to a whole taka (a fraction below 0.5 rounds down, otherwise up), through one shared function (D-026).
- Write worked examples for VAT-inclusive and VAT-exclusive prices in the docs (where the rounding difference lands).
- **API operations:** section 4.8 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M5 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-06 · Receipt data and reprint rules `[ ]`

Size M · Milestone M6 (Receipts and hardware) · Depends: M5 · Unblocks: M7 · Gate: G5

- Receipt data endpoint (shop details, BIN, bill number, cashier, QR).
- Reprint marking (`print_count`).
- `receipt_templates`.
- **API operations:** section 4.9 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M6 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-07 · Dashboard queries `[ ]`

Size S · Milestone M7 (Reports v1) · Depends: M5, M6 · Unblocks: M8, M10, M11

- Dashboard queries: today's sales, bills, average bill, profit (from cost snapshots).
- **API operations:** section 4.10 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M7 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-08 · Credit sales, gift cards and store credit payments `[ ]`

Size L · Milestone M8 (Customers, credit and loyalty) · Depends: M7 · Unblocks: M9, M12

- Credit sale in the payment flow.
- Gift card and loyalty payment methods.
- Store credit as a payment method.
- Credit-limit checks with approval.
- **API operations:** section 4.11 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M8 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-09 · Returns and refunds `[ ]`

Size M · Milestone M9 (Returns and exchanges) · Depends: M5, M8 · Unblocks: M12

- `sale_returns`, `sale_return_items`.
- Return window check.
- Refund by cash, store credit or exchange.
- Refund payments (`direction = out`).
- Return numbers.
- Approval rules.
- Refund VAT amounts come from the original sale lines and are never re-rounded (D-026).
- **API operations:** section 4.12 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M9 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-10 · Purchasing posting review and supplier payments `[ ]`

Size M · Milestone M10 (Suppliers and purchasing) · Depends: M7 · Unblocks: M12

- Review of purchasing ledger posting and approval rules.
- Supplier payment posting.
- **API operations:** section 4.13 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M10 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-11 · Promotion engine and coupons `[ ]`

Size M · Milestone M11 (Promotions and coupons) · Depends: M7 · Unblocks: M12

- `promotions`, `promotion_targets`, `coupons`, `coupon_redemptions`, `sale_promotions`.
- Evaluation engine (item, category, bill; BOGO; minimum purchase; schedule; priority; stacking).
- Coupon limits.
- **API operations:** section 4.14 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M11 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-12 · Background jobs, alerts and expenses `[ ]`

Size L · Milestone M12 (Expenses, reports and alerts) · Depends: M8, M9, M10, M11 · Unblocks: M13

- Celery and Redis introduced.
- Background jobs for expiry, low-stock, overdue-due and shift-difference alerts.
- `notifications`.
- `expense_categories` and `expenses`.
- **API operations:** section 4.15 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M12 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-13 · End-to-end acceptance suite per preset `[ ]`

Size M · Milestone M13 (Shop-type presets) · Depends: M12 · Unblocks: M14

- End-to-end acceptance test suite, one scenario per preset (requirements section 9).
- Test data builders.
- **Done when:** your part of the M13 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

### B1-14 · Security, load tests, monitoring and deployment `[ ]`

Size L · Milestone M14 (Hardening and launch readiness) · Depends: M13 · Unblocks: none · Gate: G6

- Security review and penetration testing.
- Full RLS and role audit.
- Load tests (50 branches, 10,000 bills a day, 50,000 products).
- Error logging and monitoring.
- Deployment pipeline and release without downtime.
- **API operations:** section 4.16 of `API.md`. Publish the schema for them before building.
- **Done when:** your part of the M14 exit criteria passes (see `PLAN.md`), tests are green, the OpenAPI schema is current, and the docs are updated.

---

## Log (newest last: date · task · what changed · notes for next task)

_No entries yet._