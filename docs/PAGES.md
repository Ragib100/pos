# Frontend Pages

Every planned screen of the Next.js app: what the user types or does (input), what the page shows (output), and **which part of the page calls which API operation and when**. Operation IDs such as `sales.create` are defined in `API.md`. Requirements codes are in `REQUIREMENTS.md`; tasks are in `tasks/`.

**Draft.** Routes, fields and wording are the plan; the team refines them while building. If a page needs an API operation that does not exist, add it to `API.md` first (contract first, see `CONTRACTS.md`).

## 1. How to read a page

Each page has a small header (route, who sees it, task, milestone, requirements) and a **parts table**:

| Column | Meaning |
|---|---|
| Part | A visible block of the page (form, table, dialog, button) |
| Input | What the user types, scans, selects or clicks there |
| Output | What that part shows |
| API call | The operation ID, and **when** it is called (on load, on submit, on change, and so on) |

"On load" means when the page opens. "Client only" means no API call.

**Route groups.** `/login`, `/select-context` and `/account` are shared. `/platform/**` is for platform admins. `/pos/**` is the cashier terminal (FE1). `/admin/**` is the shop owner and manager back office (FE2).

## 2. Rules for all pages

**Session and guards**
- On first load after login the app calls `auth.me` and keeps the result in memory (refetch when the window regains focus).
- No session or `AUTH_SESSION_ENDED` → go to `/login`, and return to the same page after login. A bill in progress stays in the browser.
- Session without a context (`context` is null) → go to `/select-context`.
- A platform admin goes to `/platform/shops`. Everyone else goes to the landing page of their role: terminal roles (permission `sale.create`) to `/pos`, others to `/admin/dashboard`.
- Menu items and buttons are hidden when the permission is missing in `auth.me`. This is only for convenience; the server enforces permissions.

**Shell (header on every page)**
- Shows shop, branch, register and user name from `auth.me`.
- Notification bell: `notifications.unread_count` on load and every 60 seconds (from M12).
- Language toggle (Bangla and English): client only; it sets `Accept-Language` on later calls.
- User menu: My account (`/account`), Switch shop (`/select-context`), Log out (`auth.logout`).
- Terminal pages also show the shift chip (`shifts.current` on load and after a shift opens or closes), a Lock button, and lock automatically after the idle time set in the shop settings (`auth.verify_pin` unlocks).

**Errors**
- Show `error.message`. For `VALIDATION_ERROR`, show each message beside its field from `details`.
- `APPROVAL_REQUIRED` opens the approval dialog (PG-39), then the same call is repeated with `approval_id`.
- `THROTTLED` shows a "try again shortly" message. A 5xx error shows the `request_id` so support can find it.

**Lists and forms**
- Lists use cursor paging with a "Load more" button. Filters live in the URL query string so a page can be bookmarked.
- Money and quantity inputs send decimal strings. The browser never calculates tax or discounts for a saved bill; it uses `sales.preview`.
- Names that print on receipts have an English and a Bangla field.
- Sections tied to a shop module (batch and expiry, variants, weighted items, combos, credit sales, loyalty) show only when that module is on in `enabled_modules` from `auth.me`.

---

## 3. Sign-in and account

### PG-01 · Sign in · `/login`
Who: public · Task F2-01 · M1 · NFR-05, 06

| Part | Input | Output | API call |
|---|---|---|---|
| Sign-in form | Phone or email, password | Error text for wrong details or a locked account | `auth.login` on submit (header `X-Client-Type: web`) |
| After sign-in | none | Platform admin goes to `/platform/shops`. One membership with one branch goes straight on (calls `auth.context`, then the landing page). Otherwise `/select-context`. | `auth.context` only in the single-choice case |
| Forgot password note | none | "Ask your shop owner to reset your password" (no self-service reset while SMS and email are deferred) | none |

### PG-02 · Choose shop, branch and register · `/select-context`
Who: signed in, no context yet · Task F1-01 · M1 · FR-T-06

| Part | Input | Output | API call |
|---|---|---|---|
| Shop list | Choose a shop | The user's shops with their role | `auth.memberships` on load |
| Branch and register pickers | Choose a branch, then a register (register shown only to roles that use a terminal) | Branches and registers allowed for that membership | none (from `auth.memberships`) |
| Continue button | Click | Goes to the landing page; the page remembers the last choice | `auth.context` on submit |

### PG-03 · Set up this device · `/pos/setup-device`
Who: owner or manager at the terminal · Task F1-01 · M1

| Part | Input | Output | API call |
|---|---|---|---|
| Device code field | The device code shown once when the register was created | After checking: shop, branch and register names to confirm | `auth.device_check` on submit (header `X-Device-Id`) |
| Confirm button | Click | The Next.js proxy stores the code on this device; goes to `/pos/pin` | none (stored by the proxy) |
| Current device panel | "Remove this device" | Shows the registered register, if set up | `auth.device_check` on load |

### PG-04 · PIN sign-in · `/pos/pin`
Who: cashier on a registered terminal · Task F1-01 · M1 · FR-T-06

| Part | Input | Output | API call |
|---|---|---|---|
| Staff tiles | Tap your name | The staff of this branch who have a PIN | `auth.terminal_staff` on load (header `X-Device-Id`) |
| PIN keypad | 4 to 6 digits | Wrong PIN or locked message | `auth.pin_login` on submit |
| After sign-in | none | Goes to `/pos/shift/open` if there is no open shift, otherwise `/pos` | `shifts.current` on success |
| "Use password" link | Click | Goes to `/login` | none |

### PG-05 · Screen lock · overlay on `/pos/**`
Who: signed-in cashier · Task F1-01 · M1 · NFR (screen lock)

| Part | Input | Output | API call |
|---|---|---|---|
| Lock overlay | PIN | Unlocks, or shows wrong PIN or locked message | `auth.verify_pin` on submit |
| Sign out link | Click | Back to `/pos/pin` | `auth.logout` on click |
| Lock trigger | Lock button or idle timer | The overlay covers the screen; the bill in progress stays in the browser | none |

### PG-06 · My account · `/account`
Who: any signed-in user · Task F2-01 · M1

| Part | Input | Output | API call |
|---|---|---|---|
| Profile card | none | Name, phone, shop, role | `auth.me` (already loaded) |
| Change password | Current password, new password | Success message; other sessions are signed out | `auth.change_password` on submit |
| Set PIN | Current password, new 4 to 6 digit PIN | Success message | `auth.set_pin` on submit |
| Session buttons | "Log out", "Log out everywhere" | Back to `/login` | `auth.logout`, `auth.logout_all` on click |
| Switch shop link | Click | Goes to `/select-context` | none |

---

## 4. Platform admin

### PG-07 · Shops · `/platform/shops`
Who: platform admin · Task F2-01 · M1 · FR-T-03

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Search text, status, shop type | Table of shops: name, type, status, owners, branches | `platform.shops.list` on load and when filters change |
| Shop type filter options | none | Type names | `platform.shop_types.list` on load |
| Row actions | Suspend (asks for a reason), Reactivate | Status updates in the row | `platform.shops.suspend`, `platform.shops.reactivate` on confirm |
| New shop button | Click | Goes to PG-08 | none |

### PG-08 · New shop · `/platform/shops/new`
Who: platform admin · Task F2-01 · M1 · FR-T-03, FR-CFG-01, 04

| Part | Input | Output | API call |
|---|---|---|---|
| Shop details form | Name, Bangla name, shop type, BIN, phone, email, time zone, language | Preset summary for the chosen type (modules and custom fields it turns on) | `platform.shop_types.list` on load |
| First owner form | Full name, phone, email, password | Hint when the phone already has an account (existing user is linked) | none |
| Create button | Click | Goes to PG-09 for the new shop | `platform.shops.create` on submit |

### PG-09 · Shop detail · `/platform/shops/{id}`
Who: platform admin · Task F2-01 · M1

| Part | Input | Output | API call |
|---|---|---|---|
| Shop summary | Edit name, BIN, phone, email, time zone, language | Shop, type, status, counts | `platform.shops.get` on load; `platform.shops.update` on save |
| Owners panel | "Add owner" (phone, name, email, password) | List of owners | `platform.shops.add_owner` on submit |
| Status buttons | Suspend with reason, Reactivate | Status badge | `platform.shops.suspend`, `platform.shops.reactivate` on confirm |

### PG-10 · Shop type presets · `/platform/shop-types`
Who: platform admin · Task F2-01 · M1 · FR-CFG-01

| Part | Input | Output | API call |
|---|---|---|---|
| Preset list | none | Generic, grocery, pharmacy, clothing with their modules | `platform.shop_types.list` on load |
| Edit dialog | Name, module switches, default custom fields, default settings | Saved preset | `platform.shop_types.update` on save |

---

## 5. Shop administration

### PG-11 · Shop details · `/admin/settings/shop`
Who: owner · Task F2-01 · M1 · FR-ST-01

| Part | Input | Output | API call |
|---|---|---|---|
| Shop form | Name, Bangla name, BIN, phone, email | Current values, saved message | `shop.get` on load; `shop.update` on save |

### PG-12 · General settings · `/admin/settings/general`
Who: owner · Task F2-01 (modules F2-13) · M1 · FR-ST-02 to 04, FR-CFG-02, FR-R-03

| Part | Input | Output | API call |
|---|---|---|---|
| Settings form | Rounding rule, VAT rounding (per line or per bill, once decided), service charge percent, return days, negative-stock policy, loyalty rule, digit style, bill number format | Current values | `settings.get` on load; `settings.update` on save |
| Module switches | Turn on or off: batch and expiry, variants, weighted items, combos, credit sales, loyalty | Current switches; menus change after save | `settings.modules_update` on save; then `auth.me` reload |
| Links | Click | Tax rates (PG-20), payment methods (PG-48), receipt format (PG-44) | none |

### PG-13 · Branches and registers · `/admin/branches`
Who: owner · Task F2-01 · M1 · FR-B-01, FR-T-06

| Part | Input | Output | API call |
|---|---|---|---|
| Branch table | none | Branches with code, name, status | `branches.list` on load |
| Branch dialog | Code, name, address, phone, BIN, status | Saved row | `branches.create` or `branches.update` on save |
| Register panel (for the selected branch) | none | Registers of that branch | `registers.list` (with `branch_id`) when a branch is selected |
| Add register | Name | The **device code, shown once** with a copy button | `registers.create` on submit |
| Edit register | Name, status | Updated row | `registers.update` on save |
| Replace device code | Click and confirm | A new code shown once; the old code stops working | `registers.rotate_device` on confirm |

### PG-14 · Roles · `/admin/roles`
Who: owner · Task F2-01 · M1 · section 3.3 of the requirements

| Part | Input | Output | API call |
|---|---|---|---|
| Role list | Select a role | Roles with member counts | `roles.list` on load; `roles.get` when a role is opened |
| Permission matrix | Tick permissions, tick "needs approval", set maximum discount percent, set the name | Permissions grouped by area | `permissions.list` on load; `roles.create` or `roles.update` on save |
| Delete role | Click and confirm | Error if it is a system role or has members | `roles.delete` on confirm |

### PG-15 · Members · `/admin/members`
Who: owner · Task F2-01 · M1 · FR-T-01, 02, 09

| Part | Input | Output | API call |
|---|---|---|---|
| Filters and table | Role, branch, status, search | Members with role, branches, status | `members.list` on load and on filter change |
| Add member dialog | Name, phone, email, password, role, "all branches" or chosen branches, optional PIN | Note when the phone already has an account (linked, no other shop shown) | `roles.list` and `branches.list` for the dropdowns on open; `members.create` on submit |
| Edit member | Role, branches, status (active or suspended) | Updated row | `members.get` on open; `members.update` on save |
| Remove member | Click and confirm | Error `LAST_OWNER` if this is the last owner | `members.delete` on confirm |
| Set PIN | New PIN | Success message | `members.set_pin` on submit |
| Reset password | Click and confirm | The **temporary password, shown once** | `members.reset_password` on confirm |

### PG-16 · Audit log · `/admin/audit-log`
Who: owner · Task F2-01 · M1 · FR-ST-05, NFR-17

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Actor (from the members list), action, record type, branch, date range | Rows: time, who, action, record, IP | `audit_logs.list` on load and on filter change; `members.list` for the actor dropdown |
| Row detail | Expand a row | Old and new values side by side | none (from the row) |

---

## 6. Catalog (FE2, M2)

### PG-17 · Categories · `/admin/catalog/categories`
Who: manager, owner · Task F2-02 · M2 · FR-PR-01

| Part | Input | Output | API call |
|---|---|---|---|
| Category tree | none | Nested categories | `categories.list` (with `tree=true`) on load |
| Category dialog | Parent (from the tree), name, Bangla name, sort order | Saved tree | `categories.create` or `categories.update` on save |
| Delete | Click and confirm | Category is deactivated | `categories.delete` on confirm |

### PG-18 · Brands · `/admin/catalog/brands`
Who: manager, owner · Task F2-02 · M2

| Part | Input | Output | API call |
|---|---|---|---|
| Brand table | Search text | Brands | `brands.list` on load and on search |
| Brand dialog | Name | Saved row | `brands.create` or `brands.update` on save |
| Delete | Click and confirm | Brand is deactivated | `brands.delete` on confirm |

### PG-19 · Units · `/admin/catalog/units`
Who: manager, owner · Task F2-02 · M2 · FR-PR-05

| Part | Input | Output | API call |
|---|---|---|---|
| Unit table | none | Units with "allows decimals" | `units.list` on load |
| Unit dialog | Code, name, Bangla name, allows decimals | Saved row | `units.create` or `units.update` on save |
| Delete | Click and confirm | Unit is deactivated | `units.delete` on confirm |

### PG-20 · Tax rates · `/admin/catalog/tax-rates`
Who: owner · Task F2-02 · M2 · FR-ST-02

| Part | Input | Output | API call |
|---|---|---|---|
| Tax table | Status filter | Rates with "price includes tax" | `tax_rates.list` on load |
| Tax dialog | Name, rate percent, price includes tax, status | Saved row | `tax_rates.create` or `tax_rates.update` on save |
| Delete | Click and confirm | Rate is deactivated | `tax_rates.delete` on confirm |

### PG-21 · Custom product fields · `/admin/catalog/attributes`
Who: owner · Task F2-02 (preset use F2-13) · M2 · FR-CFG-03

| Part | Input | Output | API call |
|---|---|---|---|
| Field list | none | The shop's custom fields (for example generic name, fabric) | `attribute_definitions.list` on load |
| Field dialog | Key, label, Bangla label, type (text, number, date, yes/no, choice), required, searchable, choices (for "choice"), order | Saved list | `attribute_definitions.create` or `attribute_definitions.update` on save |
| Delete | Click and confirm | Field removed from new forms | `attribute_definitions.delete` on confirm |

### PG-22 · Products · `/admin/catalog/products`
Who: manager, owner (view: stock keeper) · Task F2-02 · M2 · FR-PR-01, 02, 04

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Search text, category, brand, type, status, batch-tracked, has expiry | Table: name, SKU, category, price, status | `products.list` on load and on change; `categories.list` and `brands.list` for the dropdowns |
| Barcode field | Scan or type a barcode | Opens the matching product | `variants.list` (with `barcode`) on Enter, then goes to PG-23 |
| Export button | Format (CSV or Excel) | A file download using the current filters | `products.export` on click |
| Row actions | Open, deactivate | Row status | `products.delete` on confirm |
| Buttons | "New product", "Import" | Go to PG-23 and PG-24 | none |

### PG-23 · Product form · `/admin/catalog/products/new` and `/admin/catalog/products/{id}`
Who: manager, owner · Task F2-02 (preset forms F2-13) · M2 · FR-PR-01 to 07, FR-CFG-03

| Part | Input | Output | API call |
|---|---|---|---|
| Load | none | Product and variants (edit mode) | `products.get` on load (edit) |
| Dropdown data | none | Categories, brands, units, tax rates | `categories.list`, `brands.list`, `units.list`, `tax_rates.list` on load |
| Custom fields block | Values for the shop's custom fields (the form is built from the definitions) | Fields such as generic name or fabric | `attribute_definitions.list` on load |
| Basics | Name, Bangla name, description, category, brand, base unit, tax rate, type (simple, variable, combo, service), status | Validation messages | saved with `products.create` (new) or `products.update` (edit) |
| Tracking switches | Sold by weight, track stock, track batches, has expiry, allow negative stock | Switches shown only for enabled modules | included in the save above |
| Variants table | SKU, variant name, option values (size, colour), cost price, sale price. New product: all rows are sent together. | Variant rows with barcodes | `products.create` (new); `variants.create` for a new row and `variants.update` for an edited row (edit) |
| Barcodes | Type a barcode, or click "Generate" | Barcodes under the variant | `variants.barcodes_add`, `barcodes.generate`, `barcodes.delete` on click |
| Units and conversions (grocery) | Unit, "how many base units", pack price | Pack sizes of the variant | `variants.conversions_add`, `conversions.update`, `conversions.delete` on save or delete |
| Combo items (type combo) | Component product search, quantity | Components list | `variants.list` (search by name) while typing; `variants.combo_items_put` on save |
| Image | Choose a picture | Thumbnail | `products.image` on upload |
| Branch tab | Branch selector, branch price, minimum stock, "sold here" switch per variant | Per-branch rows | `branches.list` on load; `branch_variants.list` when a branch is chosen; `branch_variants.put` on save of a row |

### PG-24 · Import products · `/admin/catalog/import`
Who: manager, owner · Task F2-02 · M2 · FR-PR-04

| Part | Input | Output | API call |
|---|---|---|---|
| Step 1: template | Choose CSV or Excel | A template file | `products.import_template` on click |
| Step 2: check file | Upload the filled file | Rows ok and a table of errors by row and field | `products.import` (mode `dry_run`) on upload |
| Step 3: apply | Click "Import" (enabled when no errors) | Progress bar, then the summary | `products.import` (mode `apply`) on click; `jobs.get` every 2 seconds until done |

### PG-25 · Barcode labels · `/admin/catalog/labels`
Who: manager · Task F2-06 · M6 · FR-PR-03

| Part | Input | Output | API call |
|---|---|---|---|
| Item picker | Search and tick variants, copies per label | Selected list | `variants.list` (with `q`) while typing |
| Preview | none | Label sheet with name, price, barcode | `labels.data` when the selection changes |
| Print button | Click | Sends the sheet to the label printer or the browser print dialog | none (client print) |

---

## 7. Inventory (FE2, M3)

### PG-26 · Stock levels · `/admin/stock`
Who: manager, owner, stock keeper · Task F2-03 · M3 · FR-I-01, 02, 03

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Branch, category, search, "low stock only", "expiring in N days" | Table: product, batch, quantity, minimum, expiry (cost and value columns only with `product.cost.view`) | `stock.levels` on load and on change; `branches.list` and `categories.list` for dropdowns |
| Tabs | All, Low stock, Expiring | Same table with the filter preset (`low_stock=true`, or `expiring_within_days=30`) | `stock.levels` on tab change |
| Row click | Click | Goes to PG-27 | none |

### PG-27 · Stock of one item · `/admin/stock/{variant_id}`
Who: manager, owner, stock keeper · Task F2-03 · M3 · FR-I-07

| Part | Input | Output | API call |
|---|---|---|---|
| Header | none | Product and variant details | `variants.get` on load |
| Quantity by batch | none | Quantity per batch with expiry | `stock.levels` (variant filter) and `stock.batches` (variant filter) on load |
| Movement history | Movement type, date range | Ledger rows: date, type, change, cost, reference, user | `stock.movements` on load and on filter change; "Load more" passes the cursor |

### PG-28 · Opening stock · `/admin/stock/opening`
Who: manager, owner · Task F2-03 · M3 · FR-I-01

| Part | Input | Output | API call |
|---|---|---|---|
| Item picker | Search text or scan | Matching variants to add as rows | `variants.list` (search or `barcode`) while typing |
| Entry grid | Quantity, cost per unit, unit, batch number and expiry (only for batch-tracked items) | Rows with validation | none |
| Branch field | Branch (default: current) | Branch options | `branches.list` on load |
| Save button | Click | Success count; the grid clears | `stock.opening` on submit (new `client_request_id`) |

### PG-29 · Stock adjustments · `/admin/stock/adjustments`
Who: manager, owner · Task F2-03 · M3 · FR-I-04

| Part | Input | Output | API call |
|---|---|---|---|
| List | Status, branch, dates | Adjustments with reason and status | `adjustments.list` on load; `adjustments.get` when one is opened |
| Reasons | "Add reason": name, Bangla name | Reason list | `adjustment_reasons.list` on load; `adjustment_reasons.create` or `adjustment_reasons.update` on save |
| New adjustment | Reason, note, items (product search, batch, quantity change, positive or negative) | Draft saved | `variants.list` (search) while typing; `adjustments.create` on "Save draft" |
| Post button | Click; manager PIN if asked | Adjustment becomes posted; stock updates | `adjustments.post` on click (on `APPROVAL_REQUIRED`, open PG-39 then retry with `approval_id`) |

### PG-30 · Stock counts · `/admin/stock/counts` and `/admin/stock/counts/{id}`
Who: manager, stock keeper · Task F2-03 · M3 · FR-I-05

| Part | Input | Output | API call |
|---|---|---|---|
| Count list | Branch, status | Counts with status and date | `counts.list` on load |
| Start count | Branch, optional category | A count sheet with expected quantities | `counts.create` on submit |
| Count sheet | Counted quantity per row | Differences highlighted | `counts.get` on load; `counts.items_put` when a row loses focus |
| Complete button | Click; manager PIN if asked | Corrections posted; count closed | `counts.complete` on click (approval dialog on `APPROVAL_REQUIRED`) |

### PG-31 · Stock transfers · `/admin/stock/transfers`
Who: manager, owner · Task F2-03 · M3 · FR-I-06 (only if gate G3 includes branch operations in launch)

| Part | Input | Output | API call |
|---|---|---|---|
| Transfer list | Status, branches | Transfers with status | `transfers.list` on load; `transfers.get` on open |
| New transfer | Receiving branch, items with quantity | Draft transfer | `branches.list` for the dropdown; `variants.list` (search); `transfers.create` on save |
| Send | Click | Status in transit | `transfers.send` on click |
| Receive (at the receiving branch) | Quantity received per item | Status received; shortages shown | `transfers.receive` on submit |
| Cancel | Click and confirm | Status cancelled | `transfers.cancel` on confirm |

### PG-32 · Expiring and expired stock · `/admin/stock/expiring`
Who: manager, owner, stock keeper · Task F2-13 · M13 · FR-I-03, FR-PR-06 (pharmacy and food)

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Days ahead (default 30), "expired only", branch | Batches: item, batch, expiry date, days left, quantity | `stock.batches` on load and on change |
| Row action | "Write off" | Opens the adjustment form (PG-29) with the batch and a reason filled in | none (navigates with prefilled data) |

### PG-33 · Stock report · `/admin/reports/stock`
Who: manager, owner · Task F2-07 · M7 · FR-RP-06

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Branch, category, "as of" date | Item list with quantity, average cost, stock value, expiry status, and the total value | `reports.stock` on load and on change |
| Export | Format (PDF or Excel) | A file download | `reports.export` (with `name=stock`) on click (from M12) |

---

## 8. Shifts and approvals (FE1 and FE2, M4)

### PG-34 · Open shift · `/pos/shift/open`
Who: cashier · Task F1-04 · M4 · FR-SH-01, FR-T-07

| Part | Input | Output | API call |
|---|---|---|---|
| Check | none | If this user already has an open shift on this register, goes to `/pos`. If the open shift is on another register, shows where. | `shifts.current` on load |
| Opening cash form | Opening cash amount | Error if the user or register already has an open shift | `shifts.open` on submit (new `client_request_id`) |
| After opening | none | Goes to `/pos` | none |

### PG-35 · Cash in and out · dialog on `/pos`
Who: cashier · Task F1-04 · M4 · FR-SH-02

| Part | Input | Output | API call |
|---|---|---|---|
| Entry form | In or out, amount, reason | Saved; the list updates | `shifts.cash_move` on submit (on `APPROVAL_REQUIRED`, open PG-39 then retry) |
| Recent list | none | Cash movements of this shift | `shifts.cash_moves` on open |

### PG-36 · Close shift · `/pos/shift/close`
Who: cashier · Task F1-04 · M4 · FR-SH-03

| Part | Input | Output | API call |
|---|---|---|---|
| Shift summary | none | Sales, payments by method, returns, cash in and out | `shifts.summary` on load |
| Count form | Counted cash, note | After confirming: expected cash, counted cash, difference | `shifts.close` on submit |
| Z-report | Print button | The frozen Z-report, printed on the receipt printer | `shifts.z_report` on click, then client print |
| After close | "Sign out" | Back to `/pos/pin` | `auth.logout` on click |

### PG-37 · Shifts · `/admin/shifts`
Who: manager, owner · Task F2-04 · M4 · FR-SH-03

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Branch, register, cashier, status, dates | Table: cashier, register, opened, closed, difference | `shifts.list` on load and on change; `branches.list`, `registers.list`, `members.list` for dropdowns |
| Row click | Click | Goes to PG-38 | none |

### PG-38 · Shift detail · `/admin/shifts/{id}`
Who: manager, owner · Task F2-04 · M4

| Part | Input | Output | API call |
|---|---|---|---|
| Header and totals | none | Opening, expected, counted, difference | `shifts.get`, `shifts.summary` on load |
| Cash movements | none | Cash in and out with reasons | `shifts.cash_moves` on load |
| Sales of this shift | "Load more" | Bills with totals | `sales.list` (with `shift_id`) on load |
| Z-report | Print or save | The frozen report (closed shifts only) | `shifts.z_report` on click |

### PG-39 · Manager approval dialog · shared component
Who: opens on any page that needs approval · Task F2-04 · M4 · section 3.3 of the requirements

| Part | Input | Output | API call |
|---|---|---|---|
| Manager list | Choose a manager | Managers of this branch whose role can approve this action | `approvals.managers` when the dialog opens (with the `action`) |
| PIN keypad | The manager's PIN | Wrong PIN or locked message | `approvals.create` on submit |
| Result | none | Returns `approval_id` to the caller, which repeats its original request with it | none |

---

## 9. Cashier terminal: selling (FE1, M5)

### PG-40 · Sales screen · `/pos`
Who: cashier (permission `sale.create`) · Task F1-05 (hardware F1-06) · M5 · FR-S-01 to 06, 12, 13, FR-P-01 to 04, NFR-01, NFR-04, NFR-16

The main screen. Every keyboard shortcut works without a mouse (FR-S-12).

| Part | Input | Output | API call |
|---|---|---|---|
| Status bar | none | Shop, branch, register, cashier, shift chip, number of held bills | `shifts.current` on load; `sales.list` (`status=held`, `mine=true`) on load and after hold or recall; alerts bell from M12 |
| Offers banner (M11) | none | Offers active now | `promotions.active` on load |
| Search and scan box | Typed text, or a barcode scanner (ends with Enter), or a shortcut key to focus | A result list; a barcode match adds the item straight away | `variants.lookup` with `barcode` on Enter, or with `q` while typing (250 ms pause); includes `customer_id` when a customer is attached |
| Weight and unit prompt | Weight (typed or from the scale) for weighted items; unit choice for items sold in packs | The quantity, with decimals where the unit allows | none (units come from the `variants.lookup` result) |
| Stock and batch hint | Choose a batch (default: first expiring) | Stock on hand and batches with expiry; a warning when the quantity is more than stock | `stock.lookup` when an item is added to the cart |
| Cart table | Change quantity, change unit, remove a line, open the line discount | Lines with price, discount, tax and line total | `sales.preview` after every cart change (debounced) |
| Totals panel | none | Subtotal, discount, tax, service charge, rounding and grand total, plus warnings | from the latest `sales.preview` |
| Customer attach (M8) | Phone number search | Name, loyalty points, due balance; "create new" with name and phone | `customers.lookup` while typing; `customers.create` for a new customer |
| Discount dialog | Percent or amount, value, line or whole bill; price override only with `sale.price_override` | Applied discount. A discount above the role's limit (`max_discount_percent` from `auth.me`) opens PG-39 and keeps the `approval_id`. | `sales.preview` after applying |
| Coupon box (M11) | Coupon code | "Valid, estimated discount" or the reason it is not valid | `coupons.validate` on submit |
| Hold button | Click or shortcut | The cart clears and "held" is confirmed | `sales.create` (`status=held`) on click |
| Recall list | Pick a held bill | The held bill's lines load into the cart; "Discard" removes it | `sales.list` (`status=held`, `mine=true`) on open; `sales.get` on pick; `sales.discard_held` on discard |
| Pay dialog | Payment lines: method, amount, cash tendered, transaction ID (card, MFS), gift card code, loyalty points, credit | Remaining balance, and the change to give back for cash | `payment_methods.list` (`is_active=true`) on load; `gift_cards.by_code` when a code is entered |
| Complete sale | Confirm in the pay dialog | Bill number and totals; the screen resets for the next sale. Errors: out of stock, payment not complete, credit limit, shift not open. | `sales.create` (`status=completed`) on confirm; for a recalled bill `sales.update_held` (if edited) then `sales.complete` |
| Pending payment | Mark a card or MFS payment verified or failed | Payment status | `payments.update` on click |
| Receipt and drawer | none | The receipt prints and the cash drawer opens | `receipts.get` after the sale, client print and drawer, then `receipts.printed` |
| Draft protection | none | A bill in progress and its request ID survive a refresh or power cut and are cleared after the sale completes | none (browser storage) |
| Menu | Cash in/out, Close shift, Today's sales, Returns, Customers, Expenses, Receive stock, Hardware | Go to PG-35, PG-36, PG-41, PG-55, PG-54, PG-68, PG-63, PG-43 | none |

### PG-41 · Today's sales · `/pos/sales`
Who: cashier · Task F1-05 · M5 · FR-S-11, FR-S-13

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Bill number, status | The cashier's own bills for today | `sales.list` (`mine=true`, `date_from=today`) on load and on change |
| Bill detail | Click a row | Lines, payments, totals | `sales.get` on click |
| Reprint | Click | The receipt marked as a duplicate copy | `receipts.get` on click, client print, then `receipts.printed` |
| Void | Reason and manager PIN (only while the shift is open) | The bill shows as voided; stock and payments are reversed | `sales.void` on submit (approval dialog PG-39 first) |
| Fix payment | Mark a pending payment verified or failed | Updated payment | `payments.update` on click |

### PG-42 · Shift summary · `/pos/summary`
Who: cashier · Task F1-07 · M7 · FR-SH-03

| Part | Input | Output | API call |
|---|---|---|---|
| Summary cards | Refresh button | Sales count and total, totals by payment method, returns, voids, cash in and out | `shifts.summary` on load and on refresh |
| Links | Click | "Close shift" goes to PG-36 | none |

---

## 10. Receipts and hardware (M6)

### PG-43 · Hardware · `/pos/settings/hardware`
Who: owner, manager at the terminal · Task F2-06 (device logic F1-06) · M6 · NFR-11

| Part | Input | Output | API call |
|---|---|---|---|
| Printer setup | Choose the receipt printer and paper width (58 or 80 mm) | Connection status | `receipt_templates.get` on load (reads the paper width) |
| Test buttons | Test print, open cash drawer, test scanner | Pass or fail for each device | none (talks to the device) |
| Optional devices | Scale and label printer settings | Connection status | none |
| Storage | none | Settings are kept on this device only | none |

### PG-44 · Receipt format · `/admin/settings/receipt`
Who: owner · Task F2-06 · M6 · FR-S-09, 10

| Part | Input | Output | API call |
|---|---|---|---|
| Level selector | Shop default, or a branch | The template in effect and where it comes from | `receipt_templates.get` on load and on change; `branches.list` for the selector |
| Format form | Paper width, language (Bangla, English, both), header text, footer text, QR on or off, layout | A live preview using sample data | `receipt_templates.put` on save |
| Logo | Choose a picture | The logo in the preview | `receipt_templates.logo` on upload |

---

## 11. Reports, version 1, and sales back office (FE2, M5 and M7)

### PG-45 · Dashboard · `/admin/dashboard`
Who: manager, owner · Task F2-07 · M7 · FR-RP-01

| Part | Input | Output | API call |
|---|---|---|---|
| Filters | Branch, date (default today) | none | `branches.list` on load |
| Tiles and charts | none | Today's sales, bills, average bill, profit (only with `report.profit`), sales by hour, payment split | `reports.dashboard` on load and on filter change |
| Low-stock list | Click an item | Top low-stock items; click opens PG-27 | from `reports.dashboard` |
| Open shifts | Click | Who is selling now; click opens PG-38 | from `reports.dashboard` |

### PG-46 · Sales report · `/admin/reports/sales`
Who: manager, owner · Task F2-07 · M7 · FR-RP-02

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Group by (day, week, month), date range, branch, cashier, payment method | Table and chart: bills, sales, discounts, tax, returns, net | `reports.sales` on load and on change; `branches.list`, `members.list`, `payment_methods.list` for dropdowns |
| Export | Format | A file download | `reports.export` (with `name=sales`) on click (from M12) |

### PG-47 · Sales · `/admin/sales` and `/admin/sales/{id}`
Who: manager, owner (permission `sale.view_all`) · Task F2-05 · M5 · FR-S-11

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Bill number, status, branch, cashier, customer, date range | Bills with totals | `sales.list` on load and on change |
| Bill page | Open a bill | Lines, payments, promotions, customer, shift | `sales.get` on load |
| Reprint | Click | Receipt marked as a duplicate | `receipts.get`, client print, `receipts.printed` |
| Void | Reason and manager PIN (shift must be open) | Bill shows as voided | `sales.void` on submit (approval dialog PG-39 first) |

### PG-48 · Payment methods · `/admin/settings/payment-methods`
Who: owner · Task F2-05 · M5 · FR-P-01

| Part | Input | Output | API call |
|---|---|---|---|
| Method table | none | Cash, card, bKash, Nagad, Rocket, credit, gift card, loyalty, store credit, with active state | `payment_methods.list` on load |
| Method dialog | Name, Bangla name, "needs transaction ID", active | Saved row | `payment_methods.create` or `payment_methods.update` on save |

---

## 12. Customers, credit and loyalty (M8)

### PG-49 · Customers · `/admin/customers`
Who: manager, owner · Task F2-08 · M8 · FR-C-01, 02

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Search (name or phone), group, "has due", status | Table: name, phone, group, due balance, points | `customers.list` on load and on change; `customer_groups.list` for the group dropdown |
| New customer dialog | Name, phone, email, address, birthday, group, credit limit | Saved row; error if the phone already exists | `customers.create` on save |
| Row click | Click | Goes to PG-50 | none |

### PG-50 · Customer detail · `/admin/customers/{id}`
Who: manager, owner · Task F2-08 · M8 · FR-C-02, 03, FR-P-05

| Part | Input | Output | API call |
|---|---|---|---|
| Profile | Edit the customer fields, status | Balances: due, store credit, loyalty points | `customers.get` on load; `customers.update` on save; `customer_groups.list` for the group dropdown |
| Sales tab | "Load more" | The customer's bills | `customers.sales` when the tab opens |
| Ledger tab | Date range | Credit sales, payments, adjustments with running balance | `customers.ledger` when the tab opens or the range changes |
| Loyalty tab | "Adjust points": points change, reason | Points history and balance | `customers.loyalty` when the tab opens; `customers.loyalty_adjust` on submit |
| Dues tab | "Collect due": amount, payment method, transaction ID | Past collections; new balance | `due_collections.list` when the tab opens; `payment_methods.list` for the method; `due_collections.create` on submit (needs an open shift) |

### PG-51 · Customer groups and prices · `/admin/customer-groups`
Who: manager, owner · Task F2-08 · M8 · FR-C-04

| Part | Input | Output | API call |
|---|---|---|---|
| Group list | none | Groups with their discount percent | `customer_groups.list` on load; `customer_groups.get` when opened |
| Group dialog | Name, discount percent | Saved row | `customer_groups.create` or `customer_groups.update` on save; `customer_groups.delete` on confirm |
| Price list | Add products (search), special price per product | The group's special prices | `customer_groups.prices_get` when a group is opened; `variants.list` (search) while adding; `customer_groups.prices_put` on save |

### PG-52 · Gift cards · `/admin/gift-cards`
Who: manager, owner · Task F2-08 · M8 · FR-P-06

| Part | Input | Output | API call |
|---|---|---|---|
| Card list | Status, search | Cards with balance, expiry, status | `gift_cards.list` on load and on change |
| Issue card dialog | Value, optional customer, expiry, optional code | The new card and its code | `gift_cards.create` on save |
| Card detail | Click a row | Balance and history of issues and redemptions | `gift_cards.get` on open |
| Cancel or extend | Click | Updated status or expiry | `gift_cards.update` on confirm |

### PG-53 · Overdue dues · `/admin/customers/dues`
Who: manager, owner · Task F2-08 · M8 · FR-C-05

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Days overdue, minimum amount | Customers with balance, oldest due date, days overdue | `customers.dues` on load and on change |
| Row click | Click | Goes to PG-50 (Dues tab) | none |
| Note | none | "SMS reminders are not available yet" (deferred) | none |

### PG-54 · Customers at the counter · `/pos/customers`
Who: cashier · Task F1-08 · M8 · FR-S-07, FR-P-05

| Part | Input | Output | API call |
|---|---|---|---|
| Search | Phone number or name | Matches with points, due, store credit | `customers.lookup` while typing |
| New customer | Name, phone | Saved customer | `customers.create` on save |
| Collect due | Amount, payment method, transaction ID | New balance; counted in the current shift | `payment_methods.list` on open; `due_collections.create` on submit |

---

## 13. Returns and exchanges (M9)

### PG-55 · Returns at the counter · `/pos/returns`
Who: cashier (permission `return.create`) · Task F1-09 · M9 · FR-R-01, 02, 04, 05

| Part | Input | Output | API call |
|---|---|---|---|
| Find the bill | Bill number, or customer phone | The matching bills | `sales.list` (`bill_no` or `customer_id`) on submit; `customers.lookup` for the phone |
| Returnable lines | Quantity to return and condition (good or damaged) per line | Sold, already returned and returnable quantities; return-window status | `sales.returnable` when a bill is chosen |
| Refund type | Cash back, store credit, or exchange; refund method for cash back | Refund total | `payment_methods.list` on load |
| Exchange items | Add new items like on the sales screen | Price difference to pay or refund | `variants.lookup` while adding; `sales.preview` after each change |
| Confirm | Click; manager PIN if asked (outside the window or over the limit) | Return number and refund total | `returns.create` on confirm (approval dialog PG-39 on `APPROVAL_REQUIRED`) |
| Refund slip | Print button | The slip on the receipt printer | `returns.slip` on click, then client print |

### PG-56 · Returns · `/admin/returns`
Who: manager, owner · Task F2-09 · M9 · FR-R-01, 03

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Bill number, branch, date range | Returns with number, bill, refund total, type | `returns.list` on load and on change |
| Return detail | Click a row | Lines, condition, payments, approver | `returns.get` on open |
| Reprint slip | Click | The refund slip | `returns.slip` on click, then client print |
| Setting link | Click | Goes to PG-12 (return days) | none |

---

## 14. Suppliers and purchasing (M10)

### PG-57 · Suppliers · `/admin/suppliers`
Who: manager, owner · Task F2-10 · M10 · FR-PU-01

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Search, status, "we owe money" | Suppliers with the amount owed | `suppliers.list` on load and on change |
| Supplier dialog | Name, contact person, phone, email, address, BIN, status | Saved row | `suppliers.create` or `suppliers.update` on save |
| Row click | Click | Goes to PG-58 | none |

### PG-58 · Supplier detail · `/admin/suppliers/{id}`
Who: manager, owner · Task F2-10 · M10 · FR-PU-01, 05

| Part | Input | Output | API call |
|---|---|---|---|
| Header | none | Supplier details and the amount owed | `suppliers.get` on load |
| Ledger tab | Date range | Receipts, returns, payments with running balance | `suppliers.ledger` when the tab opens |
| Payments tab | "Pay supplier": amount, method, reference | Payment history; new balance | `supplier_payments.list` when the tab opens; `payment_methods.list` for the method; `supplier_payments.create` on submit |
| Orders and receipts tabs | none | This supplier's orders and goods receipts | `purchase_orders.list` and `goods_receipts.list` (with `supplier_id`) when the tabs open |

### PG-59 · Purchase orders · `/admin/purchase-orders`
Who: manager, owner · Task F2-10 · M10 · FR-PU-02

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Supplier, status, date range | Orders with total and status | `purchase_orders.list` on load and on change; `suppliers.list` for the dropdown |
| New order button | Click | Goes to PG-60 | none |

### PG-60 · Purchase order form · `/admin/purchase-orders/new` and `/admin/purchase-orders/{id}`
Who: manager, owner · Task F2-10 · M10 · FR-PU-02

| Part | Input | Output | API call |
|---|---|---|---|
| Header | Supplier, order date, expected date, note | Supplier choices | `suppliers.list` on load; `purchase_orders.get` on load (edit) |
| Item rows | Search a product, choose the unit (for example carton), quantity, cost | Row totals and order total | `variants.lookup` (search) while typing |
| Save | Click (only while draft) | Saved draft | `purchase_orders.create` or `purchase_orders.update` on save |
| Actions | Mark as sent, Print, Cancel, "Receive goods" | Status changes; the print layout; "Receive" goes to PG-61 with the lines filled in | `purchase_orders.send`, `purchase_orders.print`, `purchase_orders.cancel` on click |

### PG-61 · Goods receipt · `/admin/goods-receipts/new` and `/admin/goods-receipts`
Who: manager, stock keeper · Task F2-10 · M10 · FR-PU-03, 06, FR-PR-06

| Part | Input | Output | API call |
|---|---|---|---|
| Source | Supplier, and optionally an open purchase order | Order lines pre-filled and what is still to receive | `suppliers.list` on load; `purchase_orders.list` (open orders for the supplier); `purchase_orders.get` when an order is chosen |
| Receipt lines | Product search (for direct receipts), quantity received (full or partial), unit, cost, batch number and expiry (for batch-tracked items) | Line totals, subtotal | `variants.lookup` (search) while typing |
| Totals | Supplier invoice number, discount, purchase VAT | Subtotal, discount, VAT, total | none |
| Post button | Click | The receipt; stock and the supplier balance update | `goods_receipts.create` on submit (new `client_request_id`) |
| Receipt list | Supplier, dates | Past receipts; detail view | `goods_receipts.list` on load; `goods_receipts.get` when opened |

### PG-62 · Purchase returns · `/admin/purchase-returns`
Who: manager, owner · Task F2-10 · M10 · FR-PU-04

| Part | Input | Output | API call |
|---|---|---|---|
| List | Supplier, date range | Returns with total | `purchase_returns.list` on load; `purchase_returns.get` when opened |
| New return | Supplier, receipt (to pre-fill lines), quantity per line, reason | Return total | `suppliers.list` on load; `goods_receipts.get` when a receipt is chosen; `purchase_returns.create` on save |

### PG-63 · Receive stock at the counter · `/pos/receive`
Who: stock keeper at the terminal · Task F1-10 · M10 · FR-PU-03

A shorter version of PG-61 for quick receiving.

| Part | Input | Output | API call |
|---|---|---|---|
| Supplier and order | Supplier, optional open order | Order lines | `suppliers.list` on load; `purchase_orders.list` (open orders); `purchase_orders.get` when chosen |
| Lines | Scan or search, quantity, unit, cost, batch and expiry | Running list | `variants.lookup` on scan or typing |
| Post | Click | Success and the receipt number | `goods_receipts.create` on submit (new `client_request_id`) |

---

## 15. Promotions and coupons (M11)

### PG-64 · Promotions · `/admin/promotions`
Who: manager, owner · Task F2-11 · M11 · FR-D-01, 02, 04

| Part | Input | Output | API call |
|---|---|---|---|
| List | Status, "active on" date, search | Promotions with dates, priority, status | `promotions.list` on load and on change; `promotions.get` when opened |
| Promotion form | Name, type (percent, fixed, buy-one-get-one, minimum purchase), scope (item, category, bill), values, start and end, weekdays and time window, priority, "can combine", status | Validation messages | `promotions.create` or `promotions.update` on save |
| Targets | Products (search), categories, branches | Target list | `variants.list` (search) while typing; `categories.list` and `branches.list` on open |
| Deactivate | Click and confirm | Status inactive | `promotions.delete` on confirm |

### PG-65 · Coupons · `/admin/coupons`
Who: manager, owner · Task F2-11 · M11 · FR-D-03

| Part | Input | Output | API call |
|---|---|---|---|
| List | Promotion, status, search | Coupons with used count and expiry | `coupons.list` on load and on change; `coupons.get` when opened |
| Coupon form | Promotion, code, total limit, per-customer limit, valid from, expires, status | Saved coupon | `promotions.list` for the dropdown on open; `coupons.create` or `coupons.update` on save |
| Usage | Click a coupon | Who used it and on which bill | `coupons.redemptions` when opened |
| Deactivate | Click and confirm | Status inactive | `coupons.delete` on confirm |

---

## 16. Alerts, expenses and full reports (M12)

### PG-66 · Notifications · `/admin/notifications` and the bell menu on every page
Who: anyone with `notification.view` · Task F2-12 (terminal bell F1-12) · M12 · FR-N-01, FR-I-03

| Part | Input | Output | API call |
|---|---|---|---|
| Bell badge | none | Unread count | `notifications.unread_count` on load and every 60 seconds |
| Notification list | Filter: unread, type | Low stock, expiry, overdue dues, shift difference, with time | `notifications.list` on load and on change |
| Open an alert | Click | Marks it read, then goes to the related page (low stock to PG-26, expiry to PG-32, dues to PG-53, shift difference to PG-38) | `notifications.read` on click |
| Mark all read | Click | Badge goes to zero | `notifications.read_all` on click |

### PG-67 · Expenses · `/admin/expenses`
Who: manager, owner · Task F2-12 · M12 · FR-SH-04

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Branch, category, date range | Expenses with total | `expenses.list` on load and on change; `branches.list` for the dropdown |
| Expense dialog | Branch, category, amount, date, payment method, note | Saved row | `expense_categories.list` and `payment_methods.list` on open; `expenses.create` or `expenses.update` on save; `expenses.get` when opened; `expenses.delete` on confirm |
| Categories dialog | Name, Bangla name | Category list | `expense_categories.create`, `expense_categories.update`, `expense_categories.delete` on save or confirm |

### PG-68 · Expense from the drawer · `/pos/expenses`
Who: cashier · Task F1-12 · M12 · FR-SH-04

| Part | Input | Output | API call |
|---|---|---|---|
| Quick form | Category, amount, note ("paid from drawer" is on) | Saved; counted in the shift's expected cash | `expense_categories.list` and `payment_methods.list` on load; `expenses.create` (`from_drawer=true`) on submit |

### PG-69 · Sales breakdown report · `/admin/reports/sales-breakdown`
Who: manager, owner · Task F2-12 · M12 · FR-RP-03

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Group by (product, category, cashier, branch, payment method), date range, branch | Rows with quantity, sales, discount, profit (with `report.profit`), and totals | `reports.sales_breakdown` on load and on change |
| Export | Format | A file download | `reports.export` (with `name=sales-breakdown`) on click |

### PG-70 · Best and worst sellers · `/admin/reports/top-products`
Who: manager, owner · Task F2-12 · M12 · FR-RP-04

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Best or worst, how many, date range, branch | Ranked products with quantity and sales | `reports.top_products` on load and on change |
| Export | Format | A file download | `reports.export` (with `name=top-products`) on click |

### PG-71 · Profit and loss · `/admin/reports/profit-loss`
Who: owner (permission `report.profit`) · Task F2-12 · M12 · FR-RP-05

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Date range, branch | Sales, returns, cost of goods, gross profit, expenses by category, net profit | `reports.profit_loss` on load and on change |
| Export | Format | A file download | `reports.export` (with `name=profit-loss`) on click |

### PG-72 · Dues report · `/admin/reports/dues`
Who: manager, owner · Task F2-12 · M12 · FR-RP-07

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Customer or supplier dues, "as of" date | Parties with balance and oldest date, and a total | `reports.dues` on load and on change |
| Export | Format | A file download | `reports.export` (with `name=dues`) on click |

### PG-73 · VAT report · `/admin/reports/vat`
Who: owner (permission `report.vat`) · Task F2-12 · M12 · FR-RP-08

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Month, branch | Sales VAT by rate (taxable amount and tax), purchase VAT, net VAT payable, number of bills | `reports.vat` on load and on change |
| Export | Format | A file download for preparing the NBR form | `reports.export` (with `name=vat`) on click |

### PG-74 · Expiry report · `/admin/reports/expiry`
Who: manager, owner · Task F2-12 · M12 · FR-RP-06

| Part | Input | Output | API call |
|---|---|---|---|
| Filter bar | Days ahead, "include expired", branch | Batches with expiry, quantity and value | `reports.expiry` on load and on change |
| Export | Format | A file download | `reports.export` (with `name=expiry`) on click |

---

## 17. Data export (M14)

### PG-75 · Export shop data · `/admin/settings/data-export`
Who: owner (permission `data.export`) · Task F2-14 · M14 · NFR-07

| Part | Input | Output | API call |
|---|---|---|---|
| Choice | Tick what to include (empty means everything) | none | none |
| Start button | Click | Progress bar | `shop.export` on click; `jobs.get` every 2 seconds until done |
| Download | Click when ready | An archive of this shop's data only | none (uses `result_url` from `jobs.get`) |

---

## 18. Module visibility (shop-type behaviour)

Pages are the same for every shop type. These switches in `enabled_modules` (from `auth.me`) decide which parts appear. Shop-type presets only turn these on or off (D-007).

| Module switch | Shows on the cashier terminal (PG-40) | Shows in the back office |
|---|---|---|
| `variants` | Variant picker when a barcode or search matches several variants (size, colour) | Variants table and size and colour matrix in PG-23 |
| `weighted` | Weight prompt and scale input for weighted items | "Sold by weight" switch in PG-23 |
| `batch_expiry` | Batch choice (first-expiring by default), expiry warnings | Batch and expiry fields in PG-28 and PG-61; PG-32; PG-74 |
| `combos` | Combo lines show as one item | Combo items panel in PG-23 |
| `credit_sales` | "Credit" payment method (needs a customer) | Credit limit in PG-49 and PG-50; PG-53 |
| `loyalty` | Points shown and redeemable at payment | Loyalty tab in PG-50; loyalty rule in PG-12 |
| Units with pack sizes (always available) | Unit selector on the cart line | Units and conversions in PG-19 and PG-23 |
| Shop's custom product fields | none | Extra fields in PG-23, built from PG-21 |

## 19. Shared components

| Component | Used on | Input | Output | API call |
|---|---|---|---|---|
| Approval dialog | PG-29, PG-30, PG-35, PG-40, PG-41, PG-47, PG-55 | Manager choice, PIN | `approval_id` to the caller | `approvals.managers`, `approvals.create` |
| Product picker | PG-28, PG-29, PG-31, PG-51, PG-60 and others | Search text or barcode | A chosen variant | `variants.list`, `variants.lookup` |
| Customer picker | PG-40, PG-54, PG-55 | Phone or name | A chosen customer | `customers.lookup` |
| Branch selector | Most `/admin` filters | Choose a branch | `branch_id` for the page filter | `branches.list` |
| Date range filter | Reports, lists | From and to dates | Query values for the page | none |
| Money and quantity input | All forms | Typed number | Decimal string | none |
| Export button | Reports | Format (PDF or Excel) | File download | `reports.export` |
| Print helper | Receipts, slips, Z-report, labels | A print payload | Output on the chosen printer, or the browser print dialog | none |

## 20. API usage matrix

Every operation in `API.md` and the pages that call it. Generated from the page tables above; if you change a page, update this table too. Shared components (section 19) call their operations on behalf of the pages that use them.

| Operation | Called from |
|---|---|
| `auth.login` | PG-01 |
| `auth.device_check` | PG-03 |
| `auth.terminal_staff` | PG-04 |
| `auth.pin_login` | PG-04 |
| `auth.me` | PG-06, PG-12, PG-40 |
| `auth.memberships` | PG-02 |
| `auth.context` | PG-01, PG-02 |
| `auth.verify_pin` | PG-05 |
| `auth.set_pin` | PG-06 |
| `auth.change_password` | PG-06 |
| `auth.logout` | PG-05, PG-06, PG-36 |
| `auth.logout_all` | PG-06 |
| `approvals.managers` | PG-39 |
| `approvals.create` | PG-39 |
| `platform.shop_types.list` | PG-07, PG-08, PG-10 |
| `platform.shop_types.update` | PG-10 |
| `platform.shops.list` | PG-07 |
| `platform.shops.create` | PG-08 |
| `platform.shops.get` | PG-09 |
| `platform.shops.update` | PG-09 |
| `platform.shops.suspend` | PG-07, PG-09 |
| `platform.shops.reactivate` | PG-07, PG-09 |
| `platform.shops.add_owner` | PG-09 |
| `shop.get` | PG-11 |
| `shop.update` | PG-11 |
| `settings.get` | PG-12 |
| `settings.update` | PG-12 |
| `settings.modules_update` | PG-12 |
| `registers.rotate_device` | PG-13 |
| `members.set_pin` | PG-15 |
| `members.reset_password` | PG-15 |
| `branches.list` | PG-13, PG-15, PG-23, PG-26, PG-28, PG-31, PG-37, PG-44, PG-45, PG-46, PG-64, PG-67 |
| `branches.create` | PG-13 |
| `branches.update` | PG-13 |
| `registers.list` | PG-13, PG-37 |
| `registers.create` | PG-13 |
| `registers.update` | PG-13 |
| `roles.list` | PG-14, PG-15 |
| `roles.create` | PG-14 |
| `roles.get` | PG-14 |
| `roles.update` | PG-14 |
| `roles.delete` | PG-14 |
| `permissions.list` | PG-14 |
| `members.list` | PG-15, PG-16, PG-37, PG-46 |
| `members.create` | PG-15 |
| `members.get` | PG-15 |
| `members.update` | PG-15 |
| `members.delete` | PG-15 |
| `audit_logs.list` | PG-16 |
| `categories.list` | PG-17, PG-22, PG-23, PG-26, PG-64 |
| `categories.create` | PG-17 |
| `categories.update` | PG-17 |
| `categories.delete` | PG-17 |
| `brands.list` | PG-18, PG-22, PG-23 |
| `brands.create` | PG-18 |
| `brands.update` | PG-18 |
| `brands.delete` | PG-18 |
| `units.list` | PG-19, PG-23 |
| `units.create` | PG-19 |
| `units.update` | PG-19 |
| `units.delete` | PG-19 |
| `tax_rates.list` | PG-20, PG-23 |
| `tax_rates.create` | PG-20 |
| `tax_rates.update` | PG-20 |
| `tax_rates.delete` | PG-20 |
| `attribute_definitions.list` | PG-21, PG-23 |
| `attribute_definitions.create` | PG-21 |
| `attribute_definitions.update` | PG-21 |
| `attribute_definitions.delete` | PG-21 |
| `products.list` | PG-22 |
| `products.create` | PG-23 |
| `products.get` | PG-23 |
| `products.update` | PG-23 |
| `products.delete` | PG-22 |
| `variants.list` | PG-22, PG-23, PG-25, PG-28, PG-29, PG-31, PG-51, PG-64 |
| `variants.get` | PG-27 |
| `variants.update` | PG-23 |
| `variants.create` | PG-23 |
| `variants.lookup` | PG-40, PG-55, PG-60, PG-61, PG-63 |
| `variants.barcodes_add` | PG-23 |
| `barcodes.generate` | PG-23 |
| `barcodes.delete` | PG-23 |
| `variants.conversions_add` | PG-23 |
| `conversions.update` | PG-23 |
| `conversions.delete` | PG-23 |
| `variants.combo_items_put` | PG-23 |
| `branch_variants.list` | PG-23 |
| `branch_variants.put` | PG-23 |
| `products.image` | PG-23 |
| `products.import_template` | PG-24 |
| `products.import` | PG-24 |
| `products.export` | PG-22 |
| `jobs.get` | PG-24, PG-75 |
| `adjustment_reasons.list` | PG-29 |
| `adjustment_reasons.create` | PG-29 |
| `adjustment_reasons.update` | PG-29 |
| `adjustments.list` | PG-29 |
| `adjustments.get` | PG-29 |
| `adjustments.create` | PG-29 |
| `counts.list` | PG-30 |
| `counts.get` | PG-30 |
| `counts.create` | PG-30 |
| `transfers.list` | PG-31 |
| `transfers.get` | PG-31 |
| `transfers.create` | PG-31 |
| `stock.levels` | PG-26, PG-27 |
| `stock.lookup` | PG-40 |
| `stock.batches` | PG-27, PG-32 |
| `stock.movements` | PG-27 |
| `stock.opening` | PG-28 |
| `adjustments.post` | PG-29 |
| `counts.items_put` | PG-30 |
| `counts.complete` | PG-30 |
| `transfers.send` | PG-31 |
| `transfers.receive` | PG-31 |
| `transfers.cancel` | PG-31 |
| `shifts.current` | PG-04, PG-34, PG-40 |
| `shifts.open` | PG-34 |
| `shifts.get` | PG-38 |
| `shifts.list` | PG-37 |
| `shifts.cash_move` | PG-35 |
| `shifts.cash_moves` | PG-35, PG-38 |
| `shifts.summary` | PG-36, PG-38, PG-42 |
| `shifts.close` | PG-36 |
| `shifts.z_report` | PG-36, PG-38 |
| `payment_methods.list` | PG-40, PG-46, PG-48, PG-50, PG-54, PG-55, PG-58, PG-67, PG-68 |
| `payment_methods.create` | PG-48 |
| `payment_methods.update` | PG-48 |
| `sales.preview` | PG-40, PG-55 |
| `sales.create` | PG-40 |
| `sales.update_held` | PG-40 |
| `sales.complete` | PG-40 |
| `sales.discard_held` | PG-40 |
| `sales.list` | PG-38, PG-40, PG-41, PG-47, PG-55 |
| `sales.get` | PG-40, PG-41, PG-47 |
| `sales.void` | PG-41, PG-47 |
| `payments.update` | PG-40, PG-41 |
| `receipts.get` | PG-40, PG-41, PG-47 |
| `receipts.printed` | PG-40, PG-41, PG-47 |
| `receipt_templates.get` | PG-43, PG-44 |
| `receipt_templates.put` | PG-44 |
| `receipt_templates.logo` | PG-44 |
| `labels.data` | PG-25 |
| `reports.dashboard` | PG-45 |
| `reports.sales` | PG-46 |
| `reports.stock` | PG-33 |
| `customer_groups.list` | PG-49, PG-50, PG-51 |
| `customer_groups.create` | PG-51 |
| `customer_groups.get` | PG-51 |
| `customer_groups.update` | PG-51 |
| `customer_groups.delete` | PG-51 |
| `customers.list` | PG-49 |
| `customers.create` | PG-40, PG-49, PG-54 |
| `customers.get` | PG-50 |
| `customers.update` | PG-50 |
| `gift_cards.list` | PG-52 |
| `gift_cards.create` | PG-52 |
| `gift_cards.get` | PG-52 |
| `gift_cards.update` | PG-52 |
| `customer_groups.prices_get` | PG-51 |
| `customer_groups.prices_put` | PG-51 |
| `customers.lookup` | PG-40, PG-54, PG-55 |
| `customers.sales` | PG-50 |
| `customers.ledger` | PG-50 |
| `customers.loyalty` | PG-50 |
| `customers.loyalty_adjust` | PG-50 |
| `due_collections.create` | PG-50, PG-54 |
| `due_collections.list` | PG-50 |
| `customers.dues` | PG-53 |
| `gift_cards.by_code` | PG-40 |
| `sales.returnable` | PG-55 |
| `returns.create` | PG-55 |
| `returns.list` | PG-56 |
| `returns.get` | PG-56 |
| `returns.slip` | PG-55, PG-56 |
| `suppliers.list` | PG-57, PG-59, PG-60, PG-61, PG-62, PG-63 |
| `suppliers.create` | PG-57 |
| `suppliers.get` | PG-58 |
| `suppliers.update` | PG-57 |
| `purchase_orders.list` | PG-58, PG-59, PG-61, PG-63 |
| `purchase_orders.create` | PG-60 |
| `purchase_orders.get` | PG-60, PG-61, PG-63 |
| `purchase_orders.update` | PG-60 |
| `goods_receipts.list` | PG-58, PG-61 |
| `goods_receipts.get` | PG-61, PG-62 |
| `goods_receipts.create` | PG-61, PG-63 |
| `purchase_returns.list` | PG-62 |
| `purchase_returns.get` | PG-62 |
| `purchase_returns.create` | PG-62 |
| `suppliers.ledger` | PG-58 |
| `supplier_payments.create` | PG-58 |
| `supplier_payments.list` | PG-58 |
| `purchase_orders.send` | PG-60 |
| `purchase_orders.cancel` | PG-60 |
| `purchase_orders.print` | PG-60 |
| `promotions.list` | PG-64, PG-65 |
| `promotions.create` | PG-64 |
| `promotions.get` | PG-64 |
| `promotions.update` | PG-64 |
| `promotions.delete` | PG-64 |
| `coupons.list` | PG-65 |
| `coupons.create` | PG-65 |
| `coupons.get` | PG-65 |
| `coupons.update` | PG-65 |
| `coupons.delete` | PG-65 |
| `promotions.active` | PG-40 |
| `coupons.validate` | PG-40 |
| `coupons.redemptions` | PG-65 |
| `expense_categories.list` | PG-67, PG-68 |
| `expense_categories.create` | PG-67 |
| `expense_categories.update` | PG-67 |
| `expense_categories.delete` | PG-67 |
| `expenses.list` | PG-67 |
| `expenses.create` | PG-67, PG-68 |
| `expenses.get` | PG-67 |
| `expenses.update` | PG-67 |
| `expenses.delete` | PG-67 |
| `notifications.list` | PG-66 |
| `notifications.unread_count` | PG-66 |
| `notifications.read` | PG-66 |
| `notifications.read_all` | PG-66 |
| `reports.sales_breakdown` | PG-69 |
| `reports.top_products` | PG-70 |
| `reports.profit_loss` | PG-71 |
| `reports.dues` | PG-72 |
| `reports.vat` | PG-73 |
| `reports.expiry` | PG-74 |
| `reports.export` | PG-33, PG-46, PG-69, PG-70, PG-71, PG-72, PG-73, PG-74 |
| `shop.export` | PG-75 |
| `health.live` | system (no page) |
| `health.ready` | system (no page) |