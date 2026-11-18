# Design Decisions

A record of the choices that shape the system, with the reasons, so they are not reopened by accident.

**Status values**
- **Accepted:** decided by the project owner.
- **Proposed:** recommended during design, built into the docs, but not yet explicitly confirmed. Confirm or replace these. (None at the moment: all decisions are accepted.)
- **Superseded:** replaced by a later decision (kept for history).

**Rules for this file:** add new entries at the end with the next number. To change a decision, add a new entry and mark the old one superseded. Do not delete entries.

## Index

| ID | Decision | Status |
|---|---|---|
| D-001 | Django and PostgreSQL | Accepted |
| D-002 | Modular monolith, not microservices | Accepted |
| D-003 | Shared database, shared schema, `shop_id` plus RLS | Accepted |
| D-004 | Tenant context comes from the session; limited DB role | Accepted |
| D-005 | Global users with per-shop memberships | Accepted |
| D-006 | Shifts are per register, branch and user | Accepted |
| D-007 | Shop types are configuration | Accepted |
| D-008 | The variant is the sellable unit | Accepted |
| D-009 | Stock and balances are append-only ledgers | Accepted |
| D-010 | Completed sales are immutable, lines are snapshots | Accepted |
| D-011 | Server-generated UUIDv7 keys and per-shop idempotency | Accepted |
| D-012 | Money and quantity precision | Accepted |
| D-013 | Authentication: phone or email, per-shop PIN, no 2FA | Accepted |
| D-014 | Offline sync is deferred | Accepted |
| D-015 | SMS and email sending is deferred | Accepted |
| D-016 | In-app notifications stay in scope | Accepted |
| D-017 | Django specifics: custom user, composite keys, migrations | Accepted |
| D-018 | API and tooling stack | Accepted |
| D-019 | Next.js frontend | Accepted |
| D-020 | API authentication: server-side sessions on `auth_sessions` | Accepted |
| D-021 | Launch scope covers all shop types | Accepted |
| D-022 | Repository, modules, tracking and deployment | Accepted |
| D-023 | Next.js rendering and UI library: shadcn/ui (gate G1) | Accepted |
| D-024 | Identity rules and session lifetimes (gate G2) | Accepted |
| D-025 | Branch scope, branch pricing and negative stock (gate G3) | Accepted |
| D-026 | Bill numbers and VAT rounding (gate G4) | Accepted |
| D-027 | Delivery format and receipt printing (gate G5) | Accepted (validate on a real printer first) |
| D-028 | Out of scope for now: offline selling, restaurant mode, shift rosters | Accepted |

---

## D-001: Django and PostgreSQL
**Status:** Accepted

**Context:** The backend technology had to be chosen.

**Decision:** Django for the backend and PostgreSQL for the database. The API and tooling choices are in D-018, the frontend in D-019 and API authentication in D-020.

**Consequences:** Django apps form the module boundaries (D-002). PostgreSQL gives Row-Level Security (D-003), partial indexes, JSONB and strong transactions, which the design relies on.

---

## D-002: Modular monolith, not microservices
**Status:** Accepted

**Context:** A sale writes the bill, lines, payments, stock movements, customer ledger, loyalty and shift totals, all of which must succeed or fail together. Scale is small (up to 50 branches, about 10,000 bills a day).

**Decision:** One Django project and one deployable, split into domain apps (`accounts`, `catalog`, `inventory`, `sales`, `customers`, `purchasing`, `promotions`, `audit`). Apps interact through `services.py` interfaces, not by writing to each other's models.

**Alternatives considered:** Microservices. Rejected for now: distributed transactions and sagas for the core sale flow, broken cross-domain foreign keys, much higher operating cost for a small team.

**Consequences:** Simple transactions and one database. Later candidates to split out: notifications, reporting (read replica), print/hardware agent, background jobs. The core (sales, payments, inventory, shifts, ledgers) stays together. A test fails if one app imports another app's models directly, so the module boundaries hold.

---

## D-003: Shared database, shared schema, `shop_id` plus RLS
**Status:** Accepted

**Context:** The system must serve many shops safely, and a new shop must need no backend change.

**Decision:** One database and one schema. Every shop-owned table has `shop_id` (and `branch_id` where relevant). Isolation is enforced by PostgreSQL Row-Level Security, enabled and forced on every shop-owned table.

**Alternatives considered:**
- *Database or schema per shop:* stronger physical isolation, but many connection pools or many migrations, harder cross-shop owner reports, more operations work. Not needed at this scale.
- *One database user per shop:* adds little real security (the app server would hold every shop's credentials, so an app compromise exposes all), costs a connection pool per user, and means storing database credentials in a table. It does not address the main risk, which is an application bug that forgets a `WHERE shop_id`.

**Consequences:** Every table needs an RLS policy and an index starting with `shop_id`. Restoring one shop alone is harder, so a per-shop export is required (NFR-07). A large customer can later be moved to its own database because `shop_id` is everywhere. Build the per-shop export early.

---

## D-004: Tenant context comes from the session; limited database role
**Status:** Accepted

**Context:** RLS only protects data if the allowed shops are set correctly and the application role cannot bypass it.

**Decision:**
- The allowed shops come from the verified membership and session, never from a client-supplied `shop_id`.
- Per request, inside a transaction, the app sets `app.shop_ids` with `set_config(..., true)`.
- The application connects as a role that is not a superuser and not the table owner. Migrations run as a separate privileged role.
- Platform-admin access to shop data is limited and audited.

**Consequences:** Every request runs in a transaction. Tests must try cross-shop access and expect failure (NFR-14). Connection pooling must be compatible with per-transaction settings. Implement this first, before feature work, because retrofitting is painful.

---

## D-005: Global users with per-shop memberships
**Status:** Accepted

**Context:** One account can own several shops. One shop can have several owners. A cashier can work in different branches and shops.

**Decision:** `users` is global. Access to a shop is a `memberships` row with a role, an optional branch restriction and a per-shop PIN. Owners are memberships with an owner role. At least one active owner must remain. Roles are defined per shop. Removing a membership affects only that shop, and a shop cannot see a user's other shops.

**Consequences:** The login flow ends with choosing shop, branch and register. The session records that context (`auth_sessions`).

---

## D-006: Shifts are per register, branch and user
**Status:** Accepted

**Context:** A cashier may work a shift at Shop 1 Branch 1, then Shop 1 Branch 2, then Shop 2 Branch 3.

**Decision:** A shift belongs to one shop, branch, register and user. Moving means a new shift. Partial unique indexes allow only one open shift per user and one per register. Z-reports are frozen snapshots per shift.

**Consequences:** Cash accountability and reports are per shift. Planned rosters are not modelled (open question in the requirements).

---

## D-007: Shop types are configuration
**Status:** Accepted

**Context:** Pharmacy, grocery, clothing and other shops must share one backend. Adding a shop from the admin panel must need no backend change.

**Decision:** Shop type is a preset (`shop_types`). Behaviour is switched per shop with `shops.enabled_modules`. Extra product fields are defined per shop in `attribute_definitions` and stored in JSONB `products.attributes`. Business code has no shop-type branches.

**Consequences:** Frontends render forms from definitions. Restaurant features (tables, kitchen orders, modifiers) are not defined and would be an optional module later.

---

## D-008: The variant is the sellable unit
**Status:** Accepted

**Decision:** Every product has at least one `product_variants` row (a simple product gets one default variant). Sales, stock, prices, barcodes and branch settings reference `variant_id`.

**Consequences:** One uniform model for all shop types. Price resolution order: customer-group price, then branch price, then default price, then promotions.

---

## D-009: Stock and balances are append-only ledgers
**Status:** Accepted

**Decision:** `stock_movements`, `customer_ledger`, `supplier_ledger`, `loyalty_transactions` and `gift_card_transactions` are append-only. `stock_levels` and the cached balances on customers and suppliers are updated in the same transaction and can be rebuilt from the ledgers. `audit_logs` is also append-only.

**Consequences:** Updates and deletes on ledger tables are not allowed (enforce in the database, not only in code). Corrections are new entries. Reconciliation is possible at any time. The customer, supplier, loyalty and gift card ledgers are built together with those features.

---

## D-010: Completed sales are immutable, lines are snapshots
**Status:** Accepted

**Decision:** A completed sale is never edited. Corrections use a void or a return. A **void is allowed only while the sale's shift is still open**, needs manager approval and a reason, and writes reversing entries (reversing stock movements and payments). It never deletes rows. After the shift closes, corrections are returns. Sale lines copy name, price, cost and tax rate at the time of sale. Held bills are stored as `sales` rows with status `held` and do not affect stock.

**Consequences:** History and profit reports do not change when the catalog changes. A closed shift and its Z-report stay frozen, because nothing can be voided into them.

---

## D-011: Server-generated UUIDv7 keys and per-shop idempotency
**Status:** Accepted

**Decision:** Primary keys are UUIDv7 and are **generated by the server**. Retry safety comes from `client_request_id` on sales, which is unique **per shop** (`UNIQUE (shop_id, client_request_id)`). Client-chosen primary keys are not accepted. Bill numbers come from per-register sequences in `number_sequences`.

**Why not client-chosen keys now:** a client could submit an ID that already exists in another shop and learn from the error that it exists. A per-shop idempotency key gives the same retry safety without that leak.

**Consequences:** A retried request creates the record only once (NFR-16). Opening a shift is protected against duplicates by the one-open-shift rules (D-006).

---

## D-012: Money and quantity precision
**Status:** Accepted

**Decision:** Money is `numeric(14,2)`, quantity `numeric(14,3)` (weight-based items), percent `numeric(7,4)`. Totals are stored, not recomputed. No floating point.

**Consequences:** The VAT rounding rule is fixed in D-026 (once per bill, half up to a whole taka).

---

## D-013: Authentication: phone or email, per-shop PIN, no 2FA
**Status:** Accepted

**Decision:** Users log in with phone or email plus password. A 4 to 6 digit PIN gives quick login only on a registered terminal, belongs to one shop membership, and locks after repeated wrong attempts. Accounts lock after 5 wrong password attempts. Manager approval is a PIN entered on the cashier's screen, and both requester and approver are recorded. Two-factor authentication and its columns were removed.

---

## D-014: Offline sync is deferred
**Status:** Accepted

**Decision:** No offline selling and no sync in this version (NFR-03). This is now **out of scope until after launch, with no preparation work** (D-028). A bill in progress must still survive a page refresh or power cut (NFR-04).

**Consequences:** No sync tables, no offline tests, no offline-specific design work. If it is wanted after launch, conflict rules for stock and numbering are designed then.

---

## D-015: SMS and email sending is deferred
**Status:** Accepted

**Decision:** No SMS or email sending now. `notification_logs` stays in the design (with `mobile`, `channel`, `purpose`, `status`, `provider_ref`) but is **not created in the first migration**. Nothing else references it, so adding it later changes no other table. Customer and shop phone and email fields stay.

**Consequences:** Later work: gateway integration, message templates, per-shop opt-in settings and a background sender (a likely first service split).

---

## D-016: In-app notifications stay in scope
**Status:** Accepted

**Decision:** Staff alerts (low stock, expiry, overdue dues, shift cash difference) use the `notifications` table with read/unread state, generated by background jobs from stock, ledger and shift data. This is separate from `notification_logs` (D-015). While building, the Phase 1 dashboard shows low stock from a live query; the `notifications` table and its jobs arrive with the Phase 2 alerts.

---

## D-017: Django specifics
**Status:** Accepted

**Decision:**
- Use a custom user model from the first migration (phone or email login). Changing the user model later is painful.
- Django has no native composite foreign key. "Same shop" references are enforced with database constraints added in migrations and covered by tests.
- RLS policies, partial unique indexes and ledger protections go in migrations with `RunSQL` and a reverse.
- Run the application and migrations with separate database roles (D-004).
- Wrap stock, money and ledger changes in `transaction.atomic` and lock rows (`select_for_update`) when updating stock levels and number sequences.
- A test checks that every shop-owned table has RLS enabled and forced.

**Consequences:** Some constraints live in raw SQL migrations, which need extra care in review.

---

## D-018: API and tooling stack
**Status:** Accepted

**Context:** The API framework and supporting tools had to be chosen on top of Django and PostgreSQL (D-001).

**Decision:**

| Area | Choice |
|---|---|
| Backend | Django |
| API framework | Django REST Framework |
| Database | PostgreSQL, using the psycopg 3 driver |
| ORM | Django ORM |
| API documentation | drf-spectacular (OpenAPI schema, generated and committed) |
| Filtering | django-filter |
| Testing | Django's built-in test runner. pytest-django is optional and not adopted. |
| UUIDv7 | The standard library's `uuid.uuid7` if the chosen Python version provides it, otherwise a small library |
| Background jobs | Celery with Redis. Introduced when the first background job is needed (expiry and low-stock alerts in Phase 2, later SMS and email). Not required for the MVP. |
| CORS | Configured only if the frontend is served from a separate domain |

**Consequences:**
- Money and quantity stay `DecimalField` and are serialized as strings.
- The tenant context (D-004) must be set inside a transaction for API requests, and set explicitly in Celery tasks and management commands.
- The OpenAPI schema is regenerated and committed when the API changes, so the frontend and Claude Code have an exact API reference.
- Pagination is chosen per endpoint. Cursor pagination suits append-only tables.
- Redis is only needed once Celery is introduced.
- API authentication is covered in D-020.

---

## D-019: Next.js frontend
**Status:** Accepted

**Context:** A frontend technology had to be chosen for the POS and admin screens.

**Decision:** Next.js. The browser talks only to the Next.js origin. Next.js forwards API calls to Django (rewrites or route handlers), so the browser and the API appear to be the same origin.

**Consequences:**
- No CORS configuration is needed for the web app (CORS stays conditional, D-018).
- The session cookie (D-020) is scoped to the Next.js origin.
- The committed OpenAPI schema (D-018) can be used to generate a typed API client.
- Native clients (the Android tablet, the owner mobile app) call the Django API directly with a Bearer token.
- The rendering mode and UI libraries are not decided yet.

---

## D-020: API authentication: server-side sessions on `auth_sessions`
**Status:** Accepted

**Context:** The API needs authentication that carries the working context (shop, branch, register), can be revoked immediately, and keeps the tenant decision on the server (D-004). Access must also work for a web app now and native clients later.

**Decision:** Opaque session tokens backed by the `auth_sessions` table. Not JWT.

- **Login:** phone or email plus password creates an `auth_sessions` row. The token is a long random string. Only its SHA-256 hash is stored (`token_hash`). A fast hash is acceptable because the token is random and long.
- **Context:** a new session has no shop context. It can only list the user's memberships and select a context. Selecting shop, branch and register sets `membership_id`, `branch_id` and `register_id` on the session and rotates the token. PIN login on a registered terminal sets the context immediately.
- **Every request:** a custom DRF authentication class reads the token (cookie or `Authorization: Bearer`), hashes it and looks up the session. It rejects the request if the session is revoked or expired, or if the user, membership or shop is not active. It then sets the shop context for row-level security (D-004).
- **Revocation:** set `revoked_at`. Removing a membership, suspending a shop or disabling a user must make their sessions fail on the next request. "Log out everywhere" revokes all of a user's sessions.
- **Web transport:** an httpOnly, Secure, SameSite cookie, set through the Next.js proxy (D-019). Never localStorage. CSRF protection is required for unsafe methods because cookies are sent automatically.
- **Native transport:** the same token as a Bearer header.
- **Login protection:** failed-attempt lockout (D-013) plus DRF throttling on login and PIN endpoints.
- `auth_sessions` is a global table without shop-based row-level security, because the lookup happens before the shop is known.

**Alternatives considered:**
- *JWT:* the context would live in the token and go stale, and immediate revocation needs a blocklist, which is a session table by another name. Reconsider JWT only for third-party integrations, which would use separate API keys (not designed yet).
- *Django's built-in sessions or DRF's token authentication:* neither carries the membership, branch and register context, lists a user's sessions, or supports native Bearer clients without extra work.

**Consequences:**
- One indexed database lookup per request. Do not cache sessions at first, because caching delays revocation.
- Schema change: `refresh_token_hash` is renamed `token_hash`, `last_seen_at`, `ip_address` and `user_agent` are added, and `membership_id`, `branch_id` and `register_id` become nullable until the context is chosen (see `ER-DIAGRAM.md` and `DATA-DICTIONARY.md`).
- Session lifetime and idle-timeout values are settings, not yet chosen. The screen lock after inactivity (requirements 3.4) is a client-side lock and does not by itself end the session.

---

## D-021: Launch scope covers all shop types
**Status:** Accepted

**Context:** No first customer or shop type has been chosen. The aim is a production-ready product that can be taken to any shop, so it must support every shop type before going to market.

**Decision:** Launch (v1.0) requires Phases 1 and 2 complete for four presets: generic, grocery, pharmacy and clothing. Phases in the requirements are development stages, not launch stages. Phase 3 is post-launch unless the project owner moves items into launch. The design never targets a single shop type.

**Consequences:**
- Variants, batch and expiry, weight-based items, unit conversion, combos, returns, purchasing, customers and credit, promotions, VAT report, expenses and in-app alerts are all launch scope.
- Build order inside launch: foundation (tenancy, auth, catalog core), then the generic preset as the reference, then enable and test each module per preset. Shop types stay configuration (D-007).
- Each preset has an end-to-end acceptance scenario (requirements, section 9).
- Launch-risk item to validate with real shops: fiscal device support (EFD/SDC) may turn out to be launch-blocking. Offline selling is out of scope (D-028).

---

## D-022: Repository, modules, tracking and deployment
**Status:** Accepted

**Context:** A four-person team (two backend, two frontend) needs a shared structure so Claude Code sessions and people do not collide. The hackathon repository (a monorepo with `docs/PLAN.md`, `docs/tasks/<NAME>.md`, one Claude conversation per task) is the model, adapted for a production team.

**Decision:**

| Area | Decision |
|---|---|
| Repository | **Monorepo:** `backend/`, `frontend/`, `contracts/`, `infra/`, `loadtest/`, `scripts/`, `docs/`, `.claude/`, `.github/workflows/`, `Makefile`, `.env.example`, `CLAUDE.md` |
| Frontend | **Independent application** (`frontend/`, Next.js): its own dependencies, build, tests and deployment |
| Backend | **Independent application** (`backend/`, Django): its own dependencies, build, tests and deployment |
| Database | **Centralized migrations and schema.** All migrations live in the backend's app `migrations/` folders, with RLS policies and raw-SQL constraints in the same files. Nothing else changes the schema. One shared schema (D-003). |
| Shared code | **Shared contracts and types.** `contracts/` holds the OpenAPI `schema.yml` (generated by the backend and committed) and the TypeScript types generated from it. The frontend imports from it; the backend never imports frontend code. |
| Module structure | **Domain-based modular architecture:** one backend app per domain (D-002) plus a read-only `reports` app (no tables, reads other apps only through their `selectors.py`; shifts stay in `accounts`). In the frontend, feature folders under the `(terminal)` and `(admin)` route groups and a shared UI package. |
| Module ownership | **One designated maintainer per module** (table in `ARCHITECTURE.md`). Anyone may change a module through a pull request; its maintainer reviews it. |
| Task tracking | **GitHub Issues and a GitHub Project:** one issue per task ID, GitHub milestones M0 to M14, lane labels. The task files in `docs/tasks/` keep the specification (steps, dependencies, done-when) and a short Log of notes for the next task. They no longer hold checkboxes. |
| Branching | **Feature branches off `main`:** one short branch per task. |
| Integration | **Pull requests** with review. Nothing is pushed to `main` directly. |
| Quality checks | **Automated CI/CD:** CI on every pull request (tests, RLS coverage, import boundaries, contract up to date, lint) and automated CD from `main`. |
| Deployment | **Frontend and backend are deployed independently.** |

**Task workflow:** one task is one Claude Code session. The member types `I am <LANE>. Do <TASK-ID>.`, takes the task's GitHub issue, works on a short branch, and opens a pull request that closes the issue. Task IDs are `B1-nn`, `B2-nn`, `F1-nn`, `F2-nn`, where `nn` is the milestone number.

**Consequences:**
- `CLAUDE.md` and `docs/SETUP.md` describe the loop. B1-00 creates the GitHub Project, milestones, labels and one issue per task ID.
- Because the two applications deploy at different times, every API change must be backward compatible (new optional fields or new endpoints) or use a new path version. For an additive change, deploy the backend first, then the frontend (see `CONTRACTS.md`).
- CI uses path filters so a frontend-only change does not rebuild the backend, and the reverse.
- Claude Code does not see GitHub issue status unless the `gh` CLI is used. The task ID in the prompt and the task file are what tell it what to do.
- The load-test tool (k6 is a candidate) and the hosting setup are still open (`OPS.md`, before M14).

---

## D-023: Next.js rendering and UI library: shadcn/ui (gate G1)
**Status:** Accepted

**Decision:** The cashier terminal (`/pos/**`) is rendered on the client: it is interactive and keyboard-driven, and server rendering adds nothing to the sale screen. Admin and platform pages (`/admin/**`, `/platform/**`) may use server rendering where it helps (first load of lists and settings) and are otherwise client-rendered. All API calls go through the Next.js proxy (D-019). The UI library is **shadcn/ui**, with Tailwind CSS: the components are copied into the repository and owned by the team. Its default sizes are compact and desktop-oriented, so task F2-00 defines touch-sized variants (buttons, inputs, the PIN keypad) in the design tokens and chooses the Bangla font.

**Consequences:** F1-00 and F2-00 can start. FE1 and FE2 share one component set, owned by FE2. Terminal screens use the touch-sized variants.

---

## D-024: Identity rules and session lifetimes (gate G2)
**Status:** Accepted

**Decision:**
- One shop is one legal entity with one BIN. There is no organization layer above shops for now.
- The PIN is per shop membership (D-005).
- Customers belong to one shop and are never shared between an owner's shops.
- Session lifetime (D-020): an absolute lifetime of about one working day for terminals, and a longer one for the owner mobile app. A client-side lock after a few idle minutes locks the screen but does not end the session. The exact values are configuration: server settings per client type (set in B1-01) and the shop setting `idle_lock_minutes`.

**Consequences:** No `organizations` table. An owner with several shops under one company gets VAT reports per shop. The screen lock needs the PIN (`auth.verify_pin`).

---

## D-025: Branch scope, branch pricing and negative stock (gate G3)
**Status:** Accepted

**Decision:**
- **In launch:** branches and registers, per-branch stock, per-branch price overrides (not full price lists), restricting a member to chosen branches (FR-T-05), and per-branch reports through the branch filter (FR-B-01, FR-B-02 and the per-branch part of FR-B-03).
- **After launch:** stock transfers between branches (FR-I-06, page PG-31, `transfers.*` operations), the combined view across an owner's shops (FR-T-08 and the cross-shop part of FR-B-03), and branch overrides of settings other than the receipt template (FR-ST-06).
- **Negative stock:** the shop default is `warn_allow` (warn, then allow). The product flag `allow_negative_stock` allows it regardless of the shop policy. The pharmacy preset sets the shop policy to `block`.
- **Shift rosters** are not needed; actual shifts are enough.

**Consequences:** The transfer tables stay in the data model, but their endpoints and pages are built after launch. Requirement phases in `REQUIREMENTS.md` change: FR-T-05, FR-B-01 and FR-B-02 move to Phase 2 (launch).

---

## D-026: Bill numbers and VAT rounding (gate G4)
**Status:** Accepted

**Decision:**
- **Bill number format:** a prefix built from the branch and register codes, then a continuous number per register that never resets, for example `<branch code>-<register code>-<number>`. Sequences live in `number_sequences`.
- **VAT rounding method:** round half up to a whole taka. If the fraction is below 0.5 the amount is rounded down (floor), otherwise up (ceiling). The bill's VAT total is therefore a whole taka, stored as a number ending in `.00`.
- **VAT rounding scope:** once per bill. Each line's VAT keeps two decimals; the VAT of all lines is added, and that total is rounded with the rule above. There is no per-shop setting for this now; a per-line option can be added later if a shop needs it.

**Consequences:** B1-05 builds one shared rounding function. Returns and voids reverse the VAT amounts recorded on the original sale and are never re-rounded (D-010). B1-05 also writes worked examples for VAT-inclusive and VAT-exclusive prices (where the rounding difference lands). It is still worth checking the rule against the NBR VAT form with an accountant before launch; changing it later is a code change, not a data change. Gate G4 is fully answered.

---

## D-027: Delivery format and receipt printing (gate G5)
**Status:** Accepted (validate on a real printer first)

**Decision:** The same Next.js app is delivered as an **installable web app** on Windows desktop and Android tablet. Receipts, cash drawer opening and paper cut use raw printer commands through a **small local print service**, or WebUSB where the browser supports it. Plain browser printing is only a fallback, because it cannot reliably open a drawer or cut paper. Before building, task F1-06 runs a short test on the real printer (print, cut, drawer). If the test fails, this decision is revisited.

**Consequences:** `HARDWARE.md` (to be written) records the printer model and the test result. The local print service is a small extra deliverable owned by FE1 with BE1 support. The receipt endpoint already returns structured data, so no backend change is needed.

---

## D-028: Out of scope for now: offline selling, restaurant mode, shift rosters
**Status:** Accepted

**Decision:**
- **Offline selling and sync** are out of scope **completely** until after launch: no preparation work, no offline acceptance tests, and no launch-risk check about it. Selling needs an internet connection. If the connection drops, the bill in progress is kept in the browser and the cashier sees a clear message (NFR-04). Idempotent submissions stay, for retry safety only.
- **Restaurant mode** is not in scope.
- **Shift rosters** are not needed.

**Consequences:** Shops with unreliable internet are outside the launch audience. Revisit after launch if there is demand. Fiscal device support (EFD/SDC) is a separate question that stays in gate G6.

---

## Open items (not yet decided)

Tracked in `REQUIREMENTS.md`, section 11, and in `PLAN.md`, section 3. Still open: whether fiscal device support (EFD/SDC) is required (G6), the exact session lifetime values (set in B1-01), the receipt-printing test on a real printer (start of F1-06), and the per-requirement phase assignments.