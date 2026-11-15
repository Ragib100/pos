# Design Decisions

A record of the choices that shape the system, with the reasons, so they are not reopened by accident.

**Status values**
- **Accepted:** decided by the project owner.
- **Proposed:** recommended during design, built into the docs, but not yet explicitly confirmed. Confirm or replace these. (Currently proposed: D-022.)
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
| D-022 | Repository layout, module ownership and task workflow | Proposed |

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

**Decision:** Primary keys are UUIDv7 and are **generated by the server**. Retry safety comes from `client_request_id` on sales, which is unique **per shop** (`UNIQUE (shop_id, client_request_id)`). Client-supplied primary keys are allowed only when offline sync is built. Bill numbers come from per-register sequences in `number_sequences`.

**Why not client-chosen keys now:** a client could submit an ID that already exists in another shop and learn from the error that it exists. A per-shop idempotency key gives the same retry safety without that leak.

**Consequences:** A retried request creates the record only once (NFR-16). Opening a shift is protected against duplicates by the one-open-shift rules (D-006). When offline sync is built, allowing client-supplied keys needs a per-shop uniqueness and ownership check.

---

## D-012: Money and quantity precision
**Status:** Accepted

**Decision:** Money is `numeric(14,2)`, quantity `numeric(14,3)` (weight-based items), percent `numeric(7,4)`. Totals are stored, not recomputed. No floating point.

**Consequences:** The VAT rounding rule (per line or per bill) must be fixed in the requirements before tax code is written (open item).

---

## D-013: Authentication: phone or email, per-shop PIN, no 2FA
**Status:** Accepted

**Decision:** Users log in with phone or email plus password. A 4 to 6 digit PIN gives quick login only on a registered terminal, belongs to one shop membership, and locks after repeated wrong attempts. Accounts lock after 5 wrong password attempts. Manager approval is a PIN entered on the cashier's screen, and both requester and approver are recorded. Two-factor authentication and its columns were removed.

---

## D-014: Offline sync is deferred
**Status:** Accepted

**Decision:** No offline selling and no sync in this version (NFR-03). The design stays offline-friendly (D-011). A bill in progress must still survive a page refresh or power cut (NFR-04).

**Consequences:** No sync tables. When added later, expect conflict rules for stock and numbering to be designed then.

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
- Launch-risk items to validate with real shops: offline selling (D-014) and fiscal device support (EFD/SDC) may turn out to be launch-blocking.

---

## D-022: Repository layout, module ownership and task workflow
**Status:** Proposed

**Context:** A four-person team (two backend, two frontend) needs a shared structure so Claude Code sessions and people do not collide. The hackathon repository (a monorepo with `docs/PLAN.md`, `docs/tasks/<NAME>.md`, one Claude conversation per task) worked well and is the model.

**Decision:**
- **One repository (monorepo)** with `backend/` (Django), `frontend/` (Next.js), `infra/`, `loadtest/`, `scripts/`, `docs/`, `.claude/`, `.github/workflows/`, `Makefile`, `.env.example` and `CLAUDE.md`.
- **Docs layout:** `docs/` holds `SETUP.md`, `PLAN.md`, `REQUIREMENTS.md`, `ARCHITECTURE.md`, `CONTRACTS.md`, `DECISIONS.md`, `ER-DIAGRAM.md`, `DATA-DICTIONARY.md`, `tasks/<LANE>.md` and `results/`. Task checkboxes live only in the task files.
- **Module ownership:** BE1 owns `accounts`, `sales`, `promotions`, `audit` and infrastructure. BE2 owns `catalog`, `inventory`, `customers`, `purchasing` and `reports`. FE1 owns the cashier terminal screens. FE2 owns the admin screens and the shared design system. A member touches another lane's code only for a task listed in their file, with that lane's owner reviewing.
- **A read-only `reports` app** is added to the module list in D-002. It has no tables and reads other apps only through their `selectors.py`. `shifts` stay inside `accounts`.
- **Task workflow:** one task is one Claude Code session. The member types `I am <LANE>. Do <TASK-ID>.`, works on a **short branch per task**, and merges through a **pull request with a review** (the hackathon team worked on `main`; a production team should not). Task IDs are `B1-nn`, `B2-nn`, `F1-nn`, `F2-nn`, where `nn` is the milestone number.

**Consequences:** `CLAUDE.md` and `docs/SETUP.md` describe the loop. CI enforces the repository checks (RLS coverage, import boundaries, schema up to date). The load-test tool (k6 is a candidate) and the hosting setup are still open. Confirm or change this decision before B1-00 starts.

---

## Open items (not yet decided)

Tracked in `REQUIREMENTS.md`, section 11. The main ones that would change decisions: one shop equals one legal entity and BIN or not, per-shop versus shared PIN, sharing customers between an owner's shops, restaurant support, whether branch operations (stock transfer, branch restriction) are launch scope, whether offline mode and fiscal-device (EFD/SDC) support are launch-blocking, session lifetime and idle-timeout values, the VAT rounding rule (per line or per bill), and the Next.js rendering mode and UI libraries.
