# POS System: Requirements Document (English, refactored v2.0)

**Based on:** `POS_System_Requirements_BN.md` (Oct 4, 2026, @ODCL)
**Status:** Draft for review
**Companion documents:** `ER-DIAGRAM.md` (data model), `DATA-DICTIONARY.md` (tables and columns)

This version keeps every requirement from the original document (same FR/NFR codes) and updates it for the decisions made since:

- The system is **multi-shop and multi-branch from the start** (one backend for any shop type).
- **Offline selling and sync are out of scope until after launch.**
- **SMS and email sending are deferred.**
- **Two-factor authentication is removed.**

Section 12 lists every change from the original.

---

## Contents

1. Introduction
2. Scope decisions
3. Tenancy, users and access
4. Shop-type configuration
5. Functional requirements
6. Non-functional requirements
7. Hardware and integrations
8. Main user flow: one sale
9. Phased plan and acceptance criteria
10. Architecture constraints
11. Open questions
12. Changes from the original document

---

## 1. Introduction

The system manages sales, stock, payments and basic business records for shops in one place. The goals are fast billing, accurate stock, and a clear daily picture of the business for the owner.

- **Purpose:** A simple, fast and reliable point-of-sale (POS) system for retail shops, supermarkets, pharmacies and similar businesses. One backend serves every shop type; only the frontend and configuration differ by type.
- **Scope:** Sales and billing, payment collection, inventory, product management, customer and supplier management, reports, multiple branches, multiple shops, and user access control.
- **Out of scope:** Full accounting (general ledger) and payroll.
- **Target users:** Platform admin, shop owner, manager, cashier and stock keeper.
- **Shop types supported by configuration:** Grocery, pharmacy, clothing and generic retail. Restaurants are a stated target in the original document but have no requirements defined (see section 11).

### Key definitions

| Term | Meaning |
|---|---|
| POS | Point of sale: where a sale and its payment are completed |
| Platform admin | Operator of the whole system. Creates shops and assigns their first owner. Not a shop role. |
| Shop | A business (the tenant). Owns its own products, customers, suppliers, settings and data. Has a type (e.g. pharmacy). |
| Branch | A physical location of a shop. Has its own stock, registers and shifts. |
| Register | A POS terminal in a branch, with one cash drawer |
| Membership | A user's access to one shop: role, optional branch restriction, and PIN |
| Owner | A member of a shop with the full-control role. A shop can have several owners, and a user can own several shops. |
| SKU | A unique code for each sellable item |
| Variant | The sellable and stockable unit of a product (e.g. size and colour). Every product has at least one. |
| Shift | One cashier on one register in one branch for a period of time |
| Cash drawer | The drawer that holds cash; it opens when a bill is printed |
| VAT | Value-added tax |
| BIN | VAT registration number |
| MFS | Mobile financial service (bKash, Nagad, Rocket and similar) |
| NBR | National Board of Revenue (Bangladesh) |

---

## 2. Scope decisions

| Item | Decision |
|---|---|
| Multi-shop and multi-branch | **In scope from day one** at the data and backend level. Branch and cross-shop screens arrive in Phase 3. |
| Launch scope | **All shop types before going to market.** There is no first-customer shop type. Launch (v1.0) requires Phases 1 and 2 complete for the generic, grocery, pharmacy and clothing presets. See section 9. |
| Offline mode and sync (NFR-03) | **Out of scope until after launch** (D-028). No preparation work. Selling needs an internet connection; a bill in progress is kept in the browser if the connection drops. |
| SMS and email sending | **Deferred.** The messaging design is kept so it can be added later. Customer and shop phone and email fields stay. |
| Two-factor authentication | **Removed.** Security relies on strong passwords, lockout and per-shop PINs. |
| In-app alerts for staff | **In scope** (low stock, expiry, overdue dues, shift cash difference). |
| Restaurant mode | **Not defined.** Would be an optional module later. |
| EFD/SDC fiscal device | **Preparation only** (NFR-12). |
| E-commerce and accounting integration | **Phase 3**, export-based. |

---

## 3. Tenancy, users and access

### 3.1 Model

- A **user** is one global login identity. One person has one account, however many shops they work in or own.
- A user gets access to a shop through a **membership** that carries a role. The same user can have different roles in different shops.
- A shop can have **many owners**. A user can own **many shops**.
- A membership can be limited to specific branches, or cover all branches.
- A cashier can work **shifts in different branches and shops over time**. Each shift belongs to exactly one shop, branch, register and user.
- Removing a user from one shop does not affect their access to other shops.

### 3.2 Tenancy and access requirements

| Code | Requirement | Phase |
|---|---|---|
| FR-T-01 | One user account can belong to several shops, with a separate role in each | MVP |
| FR-T-02 | A shop can have several owners. The system must never allow a shop to be left with no active owner. | MVP |
| FR-T-03 | The platform admin can create, suspend and reactivate shops, choose the shop type, and assign the first owner | MVP |
| FR-T-04 | Shop data is strictly isolated. No user can read or change another shop's data, and this is enforced at the database level, not only in the app. | MVP |
| FR-T-05 | A membership can be limited to selected branches | Phase 2 |
| FR-T-06 | After login the user chooses shop, branch and register from the ones they are allowed to use. The choice fixes the working context of the session. | MVP |
| FR-T-07 | Only one open shift per user at a time, and only one open shift per register | MVP |
| FR-T-08 | An owner of several shops can switch between them, see each shop's reports, and see a combined report across their shops | Phase 3 |
| FR-T-09 | Removing a member from a shop only affects that shop. A shop owner cannot see which other shops a user belongs to. | MVP |
| FR-T-10 | Platform admin access to shop data is limited and recorded in the audit log | MVP |

### 3.3 Roles and permissions

Each shop has its own roles. The default roles are below. The owner can create new roles and customize permissions.

| Action | Owner | Manager | Cashier | Stock keeper |
|---|---|---|---|---|
| Create sales and bills | Yes | Yes | Yes | No |
| Give discount above the limit | Yes | Yes | Needs approval | No |
| Void a bill / process a return | Yes | Yes | Needs approval | No |
| Add products and change prices | Yes | Yes | No | No |
| Receive stock and adjust stock | Yes | Yes | No | Yes |
| Create purchase orders | Yes | Yes | No | Yes |
| View reports | All | Own branch | Own shift | Stock reports |
| Manage users | Yes | No | No | No |
| System and shop settings | Yes | No | No | No |

### 3.4 Login and security rules

- Every user logs in with their own credentials: **phone number or email, plus password**.
- A **4 to 6 digit PIN** gives quick login on a registered terminal. The PIN belongs to one shop membership. It is accepted only on registered devices and is locked after repeated wrong attempts.
- When an action needs manager approval, the manager can approve it on the cashier's screen by entering their PIN. Both the requester and the approver are recorded.
- The screen locks automatically after a period of inactivity.
- The shop context (which shop's data a request touches) is always taken from the user's verified membership and session, never from a value sent by the client.
- Sessions are server-side and can be revoked immediately, for example when a member is removed, a shop is suspended, a user is disabled or a device is lost.

---

## 4. Shop-type configuration

The backend is identical for every shop type. A shop's type only selects defaults and which parts of the product are switched on.

| Code | Requirement | Phase |
|---|---|---|
| FR-CFG-01 | Each shop type is a preset: default feature modules, custom product fields, units, tax rules and receipt template. Choosing a type when creating a shop applies the preset. | MVP |
| FR-CFG-02 | Features are switched on or off per shop: batch and expiry, variants, weight-based items, combos, credit sales, loyalty | MVP |
| FR-CFG-03 | Each shop can define its own custom product fields (for example generic name for a pharmacy, fabric for clothing) without any database or backend change | MVP |
| FR-CFG-04 | Adding a new shop from the admin panel needs no backend change; only the frontend may need adjusting for a new type | MVP |
| FR-CFG-05 | Complete presets for generic, grocery, pharmacy and clothing shops are available at launch, and each passes its end-to-end acceptance scenario (section 9) | Launch |

Examples of what a preset turns on:

| Shop type | Typical defaults |
|---|---|
| Pharmacy | Batch and expiry tracking (first-expiring, first-out), custom fields such as generic name and drug category |
| Grocery | Weight-based items, unit conversions (carton to piece), barcode scanning |
| Clothing | Variants (size, colour) with separate stock and price |
| Generic | Basic products and stock only |

---

## 5. Functional requirements

Each requirement has a code for use in development and testing. **Phase** shows the proposed delivery phase (MVP, Phase 2, Phase 3, or Deferred). Phase assignments of individual items are proposed and should be confirmed.

### 5.1 Sales and billing

| Code | Requirement | Phase |
|---|---|---|
| FR-S-01 | Add items to the cart by scanning a barcode or searching by name or SKU | MVP |
| FR-S-02 | Increase or decrease item quantity and remove items in the cart | MVP |
| FR-S-03 | Weight-based items accept decimal quantities (for example 1.25 kg) | MVP |
| FR-S-04 | Discount on an item or on the whole bill, as an amount or a percentage | MVP |
| FR-S-05 | VAT and service charge are calculated automatically. The bill's total VAT is rounded once per bill, half up to a whole taka: a fraction below 0.5 rounds down, otherwise up (D-026). | MVP |
| FR-S-06 | Hold a bill, start a new one, and recall the held bill later | MVP |
| FR-S-07 | Attach a customer to a bill (search by mobile number or create new) | Phase 2 |
| FR-S-08 | Print a receipt (58 mm or 80 mm thermal). Sending by SMS or email is **deferred** (see FR-N-02). | MVP |
| FR-S-09 | Receipt shows shop name, logo, address, phone, BIN, bill number, date and time, cashier name and a QR code | MVP |
| FR-S-10 | Receipt can be printed in Bangla and English | MVP |
| FR-S-11 | Find an old bill and reprint it, marked as a duplicate copy | MVP |
| FR-S-12 | A whole bill can be completed with keyboard shortcuts, without a mouse | MVP |
| FR-S-13 | A completed bill can be voided only while its shift is still open, with manager approval and a reason. A void reverses stock and payments with new entries and never deletes anything. After the shift closes, corrections are made by a return (FR-R-01). | MVP |

### 5.2 Payments

| Code | Requirement | Phase |
|---|---|---|
| FR-P-01 | Support cash, card and MFS (bKash, Nagad, Rocket). Credit sale (pay later) is added in Phase 2. | MVP |
| FR-P-02 | A bill can be paid with several methods (split payment) | MVP |
| FR-P-03 | After a cash payment, the change to return is shown automatically | MVP |
| FR-P-04 | Card and MFS payments store the transaction ID | MVP |
| FR-P-05 | Credit sale adds to the customer's due balance. Part or full payment can be taken later. | Phase 2 |
| FR-P-06 | Pay with a gift card or voucher, or with loyalty points | Phase 2 |

### 5.3 Product management

| Code | Requirement | Phase |
|---|---|---|
| FR-PR-01 | Add, edit and deactivate products: name (Bangla and English), SKU, barcode, category, brand, unit, cost price, sale price, VAT rate, image, and the shop's custom fields | MVP |
| FR-PR-02 | Variants of one product (size, colour) with separate stock and price | MVP |
| FR-PR-03 | The system can generate barcodes and print labels | Phase 2 |
| FR-PR-04 | Import and export many products from Excel or CSV | MVP |
| FR-PR-05 | Unit conversion (for example 1 carton = 24 pieces) | Phase 2 |
| FR-PR-06 | Batch number and expiry date (for pharmacy and food) | Phase 2 |
| FR-PR-07 | Combo or bundle products. Selling one reduces the stock of each component. | Phase 2 |

### 5.4 Inventory

| Code | Requirement | Phase |
|---|---|---|
| FR-I-01 | Stock updates in real time on every sale, return and purchase | MVP |
| FR-I-02 | Minimum stock level per item (per branch). A warning shows when stock falls below it. | MVP |
| FR-I-03 | Expiry warning before expiry (default 30 days, configurable) | Phase 2 |
| FR-I-04 | Stock adjustment for damaged, lost or stolen items, recorded with a reason | MVP |
| FR-I-05 | Stock count (stock take) with differences against the system quantity | Phase 2 |
| FR-I-06 | Transfer stock from one branch to another branch of the same shop | Phase 3 |
| FR-I-07 | Full stock movement history for every item | MVP |
| FR-I-08 | Stock valuation by weighted average cost (MVP) or FIFO (Phase 3) | MVP / Phase 3 |

### 5.5 Customer management

| Code | Requirement | Phase |
|---|---|---|
| FR-C-01 | Store customer name, mobile, address, email and birthday | Phase 2 |
| FR-C-02 | View a customer's purchase history and due balance | Phase 2 |
| FR-C-03 | Earn and spend loyalty points (for example 1 point per 100 taka) | Phase 2 |
| FR-C-04 | Customer groups (general, wholesale, VIP) with different prices or discounts | Phase 2 |
| FR-C-05 | Overdue customer dues are listed in the system. SMS reminders are **deferred** (see FR-N-02). | Phase 2 (list) / Deferred (SMS) |

Customers belong to one shop and are shared by all of its branches. They are never shared between shops.

### 5.6 Supplier and purchasing

| Code | Requirement | Phase |
|---|---|---|
| FR-PU-01 | Keep supplier details and the amount owed to each supplier | Phase 2 |
| FR-PU-02 | Create and print a purchase order (PO). Sending it to the supplier is by print or manual share for now. | Phase 2 |
| FR-PU-03 | Receive goods against a PO, fully or partially. Record the unit received, batch, expiry and cost. | Phase 2 |
| FR-PU-04 | Return goods to a supplier (purchase return) | Phase 2 |
| FR-PU-05 | Record payments to suppliers and update the amount owed | Phase 2 |
| FR-PU-06 | Record purchase discount and purchase VAT on each goods receipt, for VAT reporting | Phase 2 |

### 5.7 Returns and exchanges

| Code | Requirement | Phase |
|---|---|---|
| FR-R-01 | Find the original bill by bill number and take back all or part of the items | Phase 2 |
| FR-R-02 | Options: cash refund, store credit, or exchange for other items | Phase 2 |
| FR-R-03 | The return time limit (for example 7 days) is set in settings | Phase 2 |
| FR-R-04 | Returned items in good condition go back to stock; damaged items go to a separate loss entry | Phase 2 |
| FR-R-05 | Every return gets its own return number, printed on the refund slip | Phase 2 |

### 5.8 Discounts and promotions

| Code | Requirement | Phase |
|---|---|---|
| FR-D-01 | Run item-based or category-based offers for a set date and time | Phase 2 |
| FR-D-02 | Buy-one-get-one-free (BOGO) and minimum-purchase discount offers | Phase 2 |
| FR-D-03 | Create coupon codes with a start date, an end date and usage limits | Phase 2 |
| FR-D-04 | When several offers apply, the order in which they apply can be set | Phase 2 |

### 5.9 Shift and cash management

| Code | Requirement | Phase |
|---|---|---|
| FR-SH-01 | At the start of a shift the cashier enters the opening cash | MVP |
| FR-SH-02 | Cash in and cash out during a shift (for example petty expense) is recorded with a reason | MVP |
| FR-SH-03 | At the end of a shift, the difference between expected and counted cash is shown (Z-report) | MVP |
| FR-SH-04 | Daily shop expenses (electricity, rent, salary) are recorded by category | Phase 2 |
| FR-SH-05 | A shift is tied to one shop, branch, register and cashier. A cashier who moves to another branch or shop opens a new shift there. | MVP |

### 5.10 Reports and dashboard

| Code | Requirement | Phase |
|---|---|---|
| FR-RP-01 | Dashboard shows today's sales, number of bills, average bill value, profit and low-stock items | MVP |
| FR-RP-02 | Sales reports: daily, weekly, monthly and custom date range | MVP |
| FR-RP-03 | Sales reports by product, category, cashier, branch and payment method | Phase 2 |
| FR-RP-04 | Best-selling and worst-selling product report | Phase 2 |
| FR-RP-05 | Profit and loss report (sales minus cost minus expenses) | Phase 2 |
| FR-RP-06 | Stock report, stock value and expired-product report | MVP |
| FR-RP-07 | Customer dues and supplier dues reports | Phase 2 |
| FR-RP-08 | VAT report with the data needed for the NBR VAT form, using sales VAT and purchase VAT | Phase 2 |
| FR-RP-09 | All reports can be exported to PDF and Excel and printed | Phase 2 |

### 5.11 Multi-branch and multi-shop

| Code | Requirement | Phase |
|---|---|---|
| FR-B-01 | Several branches are run from one central shop account | Phase 2 |
| FR-B-02 | Each branch has its own stock and can have its own prices and users. The product catalog is shared across the shop's branches. | Phase 2 |
| FR-B-03 | The owner sees combined and per-branch reports for each shop, and a combined view across all shops they own | Phase 2 (per branch), Phase 3 (across shops) |

Per-branch stock, prices, member branch restriction and per-branch reports are part of launch. Stock transfers (FR-I-06) and the combined view across shops (FR-T-08) come after launch (D-025). The data model supports all of it from the start.

### 5.12 Settings

| Code | Requirement | Phase |
|---|---|---|
| FR-ST-01 | Customize shop details (name, logo, BIN, the shop's own phone and email) and receipt format | MVP |
| FR-ST-02 | Set VAT rates, currency (taka) and rounding rules | MVP |
| FR-ST-03 | Switch the interface language between Bangla and English | MVP |
| FR-ST-04 | Show dates and numbers in Bangla or English digits | MVP |
| FR-ST-05 | Audit log of every important action: who, when and what changed | MVP |
| FR-ST-06 | Settings can be set per shop, with receipt format and prices overridable per branch | Phase 3 (branch overrides) |

### 5.13 Notifications

| Code | Requirement | Phase |
|---|---|---|
| FR-N-01 | In-app alerts for staff: low stock, expiring batches, overdue customer dues and shift cash differences. Each alert can be marked read. | Phase 2 |
| FR-N-02 | **Reserved, deferred:** sending SMS or email to customers (digital receipts, due reminders, promotions), with a delivery log. The design keeps this scope ready for later. | Deferred |

---

## 6. Non-functional requirements

| Code | Area | Requirement |
|---|---|---|
| NFR-01 | Speed | An item appears in the cart within 1 second of a scan. A bill is completed and printed within 3 seconds. |
| NFR-02 | Speed | Search results appear within 1 second across 50,000 products |
| NFR-03 | Offline | **Out of scope until after launch** (D-028). Selling without internet, with automatic sync later, is not in this version and no preparation work is planned. |
| NFR-04 | Reliability | Cloud uptime of at least 99.5%. A bill in progress must not be lost on a sudden power cut or page refresh. |
| NFR-05 | Security | Passwords are stored hashed. All data in transit is encrypted with HTTPS/TLS. |
| NFR-06 | Security | An account is temporarily locked after 5 wrong login attempts. A PIN is temporarily locked after repeated wrong attempts. *(Two-factor authentication for owners was removed.)* |
| NFR-07 | Backup | Automatic daily cloud backup, kept for 30 days, with restore. A way to export one shop's data is also provided. |
| NFR-08 | Usability | A new cashier can bill after 30 minutes of training. Touchscreen-friendly with large buttons. |
| NFR-09 | Language | Full Bangla Unicode support in the interface, receipts, reports and search |
| NFR-10 | Scalability | Up to 50 branches and 10,000 bills per day without slowing down |
| NFR-11 | Platform | Runs on Windows desktop, web browser and Android tablet. A mobile app for owners. |
| NFR-12 | Compliance | Follows NBR VAT rules and is prepared for EFD/SDC device connection if needed |
| NFR-13 | Maintenance | Updates can be released without stopping sales. Error logs are stored centrally. |
| NFR-14 | Tenant isolation | Every business record belongs to one shop, and isolation is enforced by the database (row-level security), not only by application code. Automated tests must try cross-shop access and confirm it fails. |
| NFR-15 | Data integrity | A completed bill cannot be edited. Corrections are made by void or return. Stock, customer dues and supplier dues are kept as append-only ledgers. |
| NFR-16 | Duplicate safety | Submitting the same sale or payment twice (for example after a network retry) must create it only once |
| NFR-17 | Auditability | Price changes, voids, returns, role changes and platform-admin access are recorded in a tamper-resistant audit log |

---

## 7. Hardware and integrations

The system works with common market hardware and the payment services used in the country.

### Hardware

| Type | Description | Need |
|---|---|---|
| Computer or tablet | Windows 10+ PC (4 GB RAM) or Android 10+ tablet | Required |
| Thermal receipt printer | 58 mm or 80 mm, USB, Bluetooth or network | Required |
| Barcode scanner | USB or Bluetooth, 1D and 2D (QR) | Required |
| Cash drawer | Connected to the printer, opens when a bill is printed | Required |
| Label printer | For printing barcode stickers | Optional |
| Weighing scale | Sends the weight of weight-based items to the system | Optional |
| Customer display | Shows items and total to the customer | Optional |
| Card terminal | Connects to the bank's POS machine | Optional |

### Software integrations

- **MFS payments:** bKash, Nagad and Rocket merchant APIs or QR payments, with transaction verification.
- **SMS gateway:** for digital receipts, due reminders and promotions. **Deferred**, together with email.
- **E-commerce:** later, stock and order sync with online stores (for example WooCommerce, Shopify).
- **Accounting:** export of sales and expense data to accounting software.
- **REST API:** a secure API for third-party systems. Every call is scoped to the caller's shop.

---

## 8. Main user flow: one sale

A normal sale is completed in 8 steps, and the whole process should take under 30 seconds.

1. The cashier logs in with their PIN on a registered terminal. If they belong to several shops, they choose the shop and branch. They start a shift by entering the opening cash.
2. They scan or search products and add them to the cart, and adjust quantities.
3. If needed, they attach a customer by mobile number. The system shows the customer's loyalty points and dues.
4. The system applies offers, discounts and VAT automatically and shows the total.
5. The cashier chooses the payment method (cash, card, MFS, credit, or several).
   - For cash, they enter the amount received and the system shows the change.
   - For MFS or card, the transaction ID is entered or verified automatically.
6. When the bill is completed, stock is reduced, the receipt is printed and the cash drawer opens.
7. *(Deferred)* Sending the receipt by SMS or email to the customer.
8. At the end of the shift, the cashier counts the actual cash. The system produces the Z-report.

**Exceptions**
- **No internet:** Offline selling is not supported in this version. The system shows a clear message and keeps the bill in progress so it is not lost. When the connection returns, the cashier can continue.
- **Out of stock:** If an item's stock is zero, the system shows a warning and, depending on settings, blocks the sale or allows it.
- **Approval needed:** For a discount above the cashier's limit, a void or a return, a manager enters their PIN on the same screen.

---

## 9. Phased plan and acceptance criteria

The product goes to market only when it supports **all shop types** (generic, grocery, pharmacy, clothing). So the phases below are **development stages, not launch stages**:

- **Launch (v1.0) = Phase 1 + Phase 2, complete.** This includes variants, batch and expiry, weight-based items, unit conversion, combos, returns, customers and credit, purchasing, promotions, VAT report, expenses and in-app alerts, because each shop type needs some of them.
- **Phase 3 is after launch**, unless the project owner moves items into launch. Branch basics (branches, registers, per-branch stock and prices, member branch restriction, per-branch reports) are part of launch; stock transfers (FR-I-06) and the cross-shop owner view (FR-T-08) come after launch (D-025).
- **Build order inside launch:** foundation first (tenancy, auth, catalog core), then the generic preset as the reference, then enable and test each module for grocery, pharmacy and clothing.
- **Launch-risk items to check with real shops before launch:** whether fiscal device support (EFD/SDC, NBR) is required in practice. Offline selling is out of scope (D-028).

| Phase | Included | Estimated time |
|---|---|---|
| 1: MVP | Tenancy foundation (shops, owners, memberships, branches, registers), shop-type presets and custom fields, login and roles, product management, sales and billing, cash/card/MFS payments, receipt printing, basic stock, shifts and Z-report, daily sales and stock reports, audit log | 2 to 3 months |
| 2: Expansion | Customers and loyalty, suppliers and purchasing, returns, promotions, credit sales, VAT report, remaining reports, expenses, in-app alerts, batch and expiry, combos, unit conversion, stock count | 2 months |
| 3: Advanced | Multi-branch screens, stock transfer, cross-shop owner view, owner mobile app, e-commerce and accounting integration, advanced analytics, FIFO valuation | 2 to 3 months |
| Later | SMS and email, offline mode and sync, restaurant module, fiscal device integration | To be planned |

*Estimates are carried over from the original document and should be re-baselined after the technical design.*

### Acceptance criteria (MVP)

- A cashier can complete a bill of 10 items within 30 seconds.
- At the end of the day, the stock report and the sales report reconcile 100%.
- Products with Bangla names can be searched, and Bangla receipts print correctly.
- The cashier role cannot change prices or see profit in reports.
- The Z-report shows the difference between expected and counted cash correctly.
- **Tenant isolation:** A user of shop A cannot read or change shop B's data through any screen or API call, confirmed by automated tests.
- **Multi-shop cashier:** A cashier with memberships in two shops can open a shift in each (one after the other). Sales and cash are attributed to the right shop, branch, register and shift.
- **Shift rules:** The system refuses a second open shift for the same user or the same register.
- **Owners:** A shop with two owners works correctly, and removing the last owner is blocked.
- **New shop:** A platform admin can create a new shop of a chosen type from the admin panel, and its owner can start selling without any backend change.
- A bill in progress survives a page refresh or a power cut.
- Submitting the same sale twice creates only one bill.

### Acceptance criteria (launch, per shop type)

Each preset must pass its end-to-end scenario on the same backend with no shop-type code changes.

- **Generic:** create products, sell with cash and MFS, take a return, close the shift, and see correct sales and stock reports.
- **Grocery:** sell a weighed item (for example 1.25 kg), sell by the piece and by the carton from one product (unit conversion), scan a barcode, and sell a combo that reduces each component's stock.
- **Pharmacy:** receive stock with batch and expiry, sell with first-expiring-first-out, get an expiry alert, record an expired-stock adjustment, and see the expiry report.
- **Clothing:** create a product with size and colour variants and separate stock and price, sell and return a variant, and see stock per variant.
- **All types:** each shop type's custom product fields appear without any backend change.

---

## 10. Architecture constraints

These are decisions that affect how the requirements are built.

| Topic | Decision |
|---|---|
| Application structure | **Modular monolith:** one deployable backend with strict internal modules (access, catalog, inventory, sales, customers, purchasing, shifts, promotions, audit). Modules talk through defined interfaces. Separate services can be split out later where useful (notifications, reporting, print agent, background jobs). |
| Database | **One shared database and schema.** Every shop-owned table carries `shop_id` (and `branch_id` where relevant). Isolation is enforced by PostgreSQL row-level security, with a least-privilege application role. |
| Shop types | Controlled by configuration (presets, feature flags, custom fields), not by code branches |
| Stock and balances | Append-only ledgers; cached balances can be rebuilt from them |
| Identifiers | Time-ordered UUIDs generated by the server. Retries are safe through a per-shop idempotency key on sales. |
| Reports | Queries over the main tables. Summary tables or read replicas are added only if volume requires it. |
| Frontend | Next.js. It proxies API calls to the backend. |
| API authentication | Server-side sessions with opaque tokens (database-backed, revocable). Not JWT. |
| Deferred features | Out of scope for now: offline selling and sync. Designed but not built: SMS and email sending |

---

## 11. Open questions

**Answered** (recorded in `DECISIONS.md`):

1. **Legal entity.** One shop is one legal entity with one BIN (D-024).
2. **Customers across shops.** Kept separate per shop (D-024).
3. **PIN scope.** Per shop (D-024).
4. **Branch pricing.** A per-branch price override is enough (D-025).
5. **Negative stock.** Default warn and allow, with a per-product override; the pharmacy preset blocks (D-025).
6. **Bill number format.** Branch and register prefix, continuous, no reset (D-026).
7. **Rosters.** Not needed (D-028).
8. **Restaurant support.** Out of scope for now (D-028).
9. **UI library.** shadcn/ui (D-023).
10. **VAT rounding.** Once per bill, half up to a whole taka (D-026).

**Still open:**

11. **Phase assignments.** Please confirm the proposed phase for each requirement in section 5.
12. **Fiscal device.** Is EFD/SDC support required for the target shops (gate G6)?
13. **Receipt printing.** Validate the planned approach on a real printer at the start of F1-06 (D-027).

---

## 12. Changes from the original document

| Area | Change |
|---|---|
| Structure | Reorganized into numbered sections. Added tenancy, shop-type configuration and architecture sections. |
| Multi-shop | New FR-T-01 to FR-T-10: users in many shops, many owners per shop, platform admin, strict tenant isolation, shop and branch context at login |
| Shop types | New FR-CFG-01 to FR-CFG-04: presets, feature switches, custom product fields, no backend change for a new shop |
| Roles | Roles are now per shop. Login by phone or email and password, with a per-shop PIN on registered devices. |
| Two-factor | **Removed** from NFR-06 |
| Offline | NFR-03 and the offline acceptance test **deferred**. Replaced by draft-bill protection (NFR-04) and duplicate-safe submissions (NFR-16). |
| SMS and email | FR-S-08, FR-C-05 and flow step 7 **deferred**. New reserved FR-N-02. |
| Notifications | New FR-N-01: in-app staff alerts |
| Shop details | FR-S-09 and FR-ST-01 now include the shop's own phone and email |
| Purchasing | New FR-PU-06 (purchase discount and VAT), and unit tracking in FR-PU-03 |
| Returns | New FR-R-05 (return numbers) |
| Shifts | New FR-SH-05 (shift tied to one shop, branch, register and cashier) |
| Reports | FR-RP-08 now uses sales VAT and purchase VAT |
| Settings | New FR-ST-06 (per-shop settings, branch overrides) |
| Multi-branch | FR-B-01 to FR-B-03 clarified: shared catalog, branch-level stock, prices and users, cross-shop owner view |
| Non-functional | New NFR-14 (tenant isolation), NFR-15 (data integrity), NFR-16 (duplicate safety), NFR-17 (auditability). NFR-07 adds per-shop export. |
| Phases | Added a phase column per requirement. Phase 1 now includes the tenancy foundation, and no longer includes offline mode. SMS moved out of Phase 2 into Deferred. |
| Acceptance | Removed the offline test. Added tenant isolation, multi-shop cashier, shift rules, owners, new-shop, draft-bill and duplicate tests. |
| Launch scope | Added: launch requires Phases 1 and 2 for all shop types (FR-CFG-05, per-shop-type acceptance scenarios). Phases are development stages, not launch stages. |
| Voids and IDs | Added FR-S-13 (void only while the shift is open). Identifiers are generated by the server, with a per-shop idempotency key for retry safety. |
| Gate decisions | Gates G1 to G5 answered and offline taken fully out of scope (D-023 to D-028). Branch basics moved into launch (FR-T-05, FR-B-01, FR-B-02); stock transfers (FR-I-06) and the cross-shop view (FR-T-08) stay after launch. |
| VAT and UI | VAT is rounded once per bill, half up to a whole taka (FR-S-05, D-026), and the UI library is shadcn/ui (D-023). |