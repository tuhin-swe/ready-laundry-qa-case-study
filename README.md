# Ready Laundry — QA testing, bug fixing and test automation

**Arifur Rahman Tuhin** · Software QA Engineer · [@tuhin-swe](https://github.com/tuhin-swe)
**Project:** Ready Laundry, a multi-vendor laundry platform (Laravel 10): customer website, admin panel, shop panel, and the API used by the customer, shop and driver apps.
**Period:** 25 Oct – 9 Nov 2026

I tested the whole platform as every kind of user, wrote each issue up in a bug report, and fixed the issues one by one on a separate `qa` branch. Every fix was checked in a real browser before I marked it fixed. At the end I wrote an automated end-to-end test suite (Playwright, 134 tests) that plays each role, and a short load test (k6). Both pass.

> The bug report itself is a private Google Sheet with four tabs: `Summary`, `Website` (WEB-xx), `Admin Panel` (AD-xx) and `Shop Panel` (SHOP-xx). The IDs below point to its rows. This page shows my work; no company data, code or credentials are included. The screenshots use demo data from a local copy.

---

## In numbers

| | |
|---|---|
| Items logged and fixed | **184**: Website 54, Admin Panel 71, Shop Panel 59 |
| By type | **136 bugs** · **46 new features** · 2 design improvements |
| Severity | Critical 21 · Major 75 · Minor 84 · Trivial 2 |
| Priority | High 75 · Medium 86 · Low 23 |
| Code changes | **212 commits** on the `qa` branch · 366 files |
| Languages | English, Bangla and Arabic: 2,336 panel texts and the whole website |
| Automated tests | **134 end-to-end tests, all passing** (guest, customer, shop owner, shop staff, admin, driver app API) |
| Load test | 50 visitors at once plus the app API: **0 errors**, 95% of pages in under 0.25 s |

All 184 items are marked *QA: Retest / DEV: Fixed* and wait for final verification. Every row has the same columns: ID, module, title, description, steps, test data, expected and actual result, severity, priority, environment, statuses, who fixed it, attachment, reporter and type.

## How I worked

1. **Test** a page or flow as each kind of user: guest, customer, shop owner, shop staff, admin, and the driver app through its API.
2. **Report** the issue in the sheet with clear steps and the exact test data.
3. **Fix** it on the `qa` branch, never on `main`, `dev` or the live branch. Each fix is its own commit named after the bug ID, for example `fix(SHOP-58): …`.
4. **Verify** it in headless Chrome at desktop and phone widths, in light and dark mode, and in each language. Anything that moves money (orders, driver pay, commission, payments) I tested inside a database transaction, rolled back, and confirmed the data was unchanged.
5. **Automate** the main flows (below), so a later change that breaks them is caught at once.
6. **Update** the bug row: *QA Status = Retest, DEV Status = Fixed*.

---

## Automated testing

I wrote the suite in TypeScript with Playwright. The test code stays with the project, so it is not published here; this section shows what it covers and the results.

![Playwright report: 134 passed](screenshots/test-01-playwright-report.jpg)

### End-to-end tests (Playwright) — 134 tests, all passing

| Role | Tests | What is covered |
|---|---:|---|
| Guest | 21 | Every public page opens, Bangla home page, home page on a phone. Dashboard pages ask to sign in. The public shop map shows no emails or phones. A wrong password is refused. A driver cannot sign in to the dashboard. |
| Customer | 11 | Every account page. **A real order from start to end**: service → cart → checkout (address, pickup and delivery times, cash on delivery) → the order is in My orders with the shop's order number. Order counts add up. Search finds the order. Another customer's order never shows. |
| Shop owner | 39 | Dashboard (and on a phone). **An order moved one step at a time**: only the next step is offered. Skipping a step through the link is refused. Picked up asks for a driver in a popup. Assigning a driver does not change the status. No going back, and no Cancel after pickup. Another shop's order cannot be touched. **A POS cash sale** with change and a receipt. The online payment page. The driver page uses the admin's payout methods. The weekly time slots form saves. A staff member is added. Every menu page opens. |
| Shop staff | 9 | The staff member added above signs in and sees only orders. No money figures and no Products, Drivers or Staff menu. Closed pages show the "no access" page. Order statuses cannot be changed. |
| Admin | 47 | Dashboard (and on a phone). **In Bangla the dashboard is fully Bangla**, dates too. Driver payout methods: add, refuse to hide them all, remove. Every one of 41 menu pages opens. |
| Driver app API | 7 | Sign in, wrong password, 401 without a token, account, payment requests with the admin's methods, validation, orders list. |

**Every test also fails on any script error in the browser or any 5xx answer from the server.** Several bugs below were found this way, not by the thing the test was looking at.

**The tests never leave test data behind.** Before the run, the whole database is dumped. The test accounts get one known password, and phone pushes to them are switched off. After the run, the dump is loaded back: orders, staff, passwords, everything. A failed setup restores at once. The test server sends mail to the log, so no customer gets an email. I checked after each run that the tables were byte-for-byte the same.

### Bugs the automated tests found (and I fixed)

| ID | Found by | What was wrong |
|---|---|---|
| WEB-54 | customer test | **My orders:** "Active" counted only *Processing*, so Pending, Confirm, Picked up and On Going orders were in no box (9 total = 0 + 6 + 0). The website showed **#ORD-000056** while the shop sees **#MV000056** for the same order. It also had an access-control gap, fixed and not described here. |
| SHOP-59 | shop test | **POS online payment page** (used by the shop app): a 500 error, a slow lookup that read far more data than it needed, and a script error whenever Stripe posted a message. |
| AD-70 | guest test | A signed-out dashboard link (a bookmark) opened the **customer website** instead of the sign-in, and after signing in the page asked for was lost. |
| AD-71 | admin test | Settings > Social link threw `$ is not defined` on every visit. |

### Load test (k6) — passed

The k6 script runs for about 2 minutes. It only reads: no orders, no sign-ins.

| Scenario | Load | Result |
|---|---|---|
| Website | up to **50 visitors at once** browsing home, services, nearest stores, FAQ, contact, sign-in | **2,202 pages, 0 errors**, 95% under **211 ms** |
| Customer app API | 54 requests a minute (services, shops, banners, areas, …) | **46 calls, 0 errors**, 95% under **216 ms** |
| API rate limit | 75 quick calls | after 60 calls in a minute the API answers **429 Too Many Requests**: the protection works |

![k6: load, response time and errors over the run](screenshots/test-02-k6-overview.jpg)

![k6: summary tables](screenshots/test-03-k6-summary.jpg)

The charts show the visitors rising to 50 and falling again. The response time stays around 100–200 ms the whole time, and the failed-request line stays at 0%.

The test server was the project on Apache with OPcache and debug off, like production. On the first try about 4% of pages failed with "Access denied for user forge". I traced this to the test server, not the app. On Windows, Apache runs PHP in threads, and the `.env` values set with `putenv` got mixed up between requests that ran at the same time. Production avoids this with `php artisan config:cache`. I gave the test server the values per request and the errors went to zero.


---

## New features I built

| Feature | IDs | What it does |
|---|---|---|
| **A real POS** | SHOP-30, SHOP-31, SHOP-48 … SHOP-52 | Product search and filters. A customer search or a walk-in customer. Discounts. Cash with change, the shop's own gateways, or pay later. Part payment now, the rest at handover. Clothes left at the shop get drop-off and ready dates, pickup and delivery charges, and are collected at the counter or delivered by a driver. 80 mm receipt. Every sale has its own page with steps: Received → Washing → Ready → Done. |
| **Orders move one step at a time** | SHOP-53, SHOP-55, SHOP-58 | Pending → Confirm → Picked up → Processing → On Going → Delivered. The menu offers only the next step. Picked up and Delivered first ask for a driver in a popup. Assigning a driver no longer marks the order done. No going back, no skipping. Cancel is allowed only before pickup. The shop app API follows the same rule. |
| **Drivers ask for their pay** | SHOP-27 … SHOP-29, SHOP-42, SHOP-56 | Each driver has pickup and delivery pay, cash held and pay due. In the driver app a driver asks the shop for their pay, choosing the method and number. The shop pays it (with the transaction ID) or turns it down with a reason, and the driver is notified. |
| **Payout methods set by the admin** | AD-68 | The app is sold in many countries, so Cash / bKash / Nagad were not left hard-coded. The admin keeps the list (add M-Pesa, PayPal, …, hide or remove). Shops and the driver app use it. A live preview shows what drivers will see. |
| **Website in Bangla and Arabic** | WEB-42, AD-42, AD-64, AD-65, AD-69, WEB-53 | The website and the whole panel in English, Bangla and Arabic. This covers 2,336 panel texts, placeholders, roles, statuses, dates and the website content the admin writes. |
| **Weekly time slots** | SHOP-57 | Pickup and delivery times on one weekly form: open/closed per day, hours, orders per slot, "use Saturday's times for every day". It shows how many orders each day can take. |
| **Shop staff with permissions** | SHOP-26, SHOP-43 | The owner adds staff and ticks exactly what each may do. Staff see no links to pages they may not open. |
| **Dashboards** | AD-27, AD-62, AD-63 | New admin and shop dashboards: correct figures, charts, what needs attention, recent orders, next 7 days. The admin sees their own income (commission and plans). |
| **Location** | WEB-39 … WEB-41, SHOP-33 … SHOP-35, SHOP-44, SHOP-45, AD-59 … AD-61 | Visitor location and nearest stores, delivery areas drawn on a map, shop location from where the shop is now, the admin's map of all shops. |
| **Dark mode** | AD-43, WEB-37 | Light and dark mode for the panel and the website. |
| **Messages and notifications** | AD-57, WEB-46, AD-66 | Contact form messages reach the admin, customers get notifications on the website, and the admin is told when a shop asks for a new service. |

| Admin: driver payout methods, with a live preview | POS sale page with its steps |
|---|---|
| ![Payout methods](screenshots/admin-11-payout-methods.jpg) | ![POS sale](screenshots/shop-22-pos-sale-page.jpg) |
| **Admin dashboard in Bangla** | **Admin dashboard in Arabic** |
| ![Bangla](screenshots/admin-12-dashboard-bangla.jpg) | ![Arabic](screenshots/admin-13-dashboard-arabic.jpg) |
| **A driver asks for their pay** | **Weekly pickup times** |
| ![Payment request](screenshots/shop-20-driver-payment-request.jpg) | ![Times](screenshots/shop-21-pickup-times.jpg) |

## How I made the user experience better

- **One clear next action.** The order menu shows only the next step, the driver page has two amounts with one action each, and the POS shows the next step as one button.
- **Ask, don't guess.** Picked up and Delivered ask for a driver in a popup. Cancelling an order or removing a payout method asks to confirm.
- **Messages that say what to do.** For example "The next step for this order is Picked up", "Assign a driver for the pickup first" or "Keep at least one method on". No bare errors.
- **The same numbers everywhere.** One money rule for every report and wallet. The customer sees the shop's order number. The dashboard counts orders on the day they were delivered.
- **Nobody lands on the wrong page.** Signed-out dashboard links go to the sign-in and back to the page. Customers go to the website sign-in. Staff and drivers get a clear "not for this account" page instead of a bare 403.
- **Works on a phone, in the dark and in every language.** I checked pages at 390 px wide, in dark mode and in Bangla and Arabic. Placeholders, dates and roles are translated too.
- **Calm interface.** No sliding panels or jumping layouts. Rows open pages, forms start under the header, and pages paint once instead of flashing.
- **Forms that help.** A red * on every required field. One image upload box that shows the saved image. Quick choices like "+1 day" and "use Saturday's times for every day".

---

## Highlights from the earlier work

### Security issues found and fixed

I found and fixed **14 security issues**: pages and API calls that worked without signing in, one shop or driver able to see or change another's data, customers able to reach other customers' orders and invoices, prices trusted from the browser, and payments marked paid before they were verified. The exact pages are left out here on purpose.

### Shop panel: a POS a shop can really use (SHOP-30, SHOP-31)

Before the fix, any product without a discount price was added at **$0**, totals came from the browser, and there was no change, receipt or walk-in customer. The new POS has:

- product search and filters;
- a customer search, or a walk-in customer;
- discounts as an amount or a percent;
- cash with the change to give back, the shop's own payment gateways (Stripe and others), or pay later;
- an 80 mm receipt, and a sales page with the money still to collect.

| POS | Sale saved, change to give |
|---|---|
| ![POS](screenshots/shop-02-pos.jpg) | ![Sale done](screenshots/shop-14-pos-sale-done.jpg) |
| **Paying by Stripe inside the POS** | **POS in dark mode** |
| ![Stripe](screenshots/shop-17-pos-stripe.jpg) | ![POS dark](screenshots/shop-03-pos-dark.jpg) |

| Receipt (cash) | Receipt (pay later) | POS on a phone |
|---|---|---|
| ![Receipt](screenshots/shop-15-receipt-cash.jpg) | ![Unpaid receipt](screenshots/shop-16-receipt-unpaid.jpg) | ![Phone](screenshots/shop-18-pos-phone.jpg) |

![POS sales](screenshots/shop-04-pos-sales.jpg)

### Drivers: pickup, delivery, pay and cash (SHOP-27 … SHOP-29, SHOP-37 … SHOP-39)

- The shop sets a **pickup pay** and a **delivery pay** for each driver. The pickup pay is booked when the clothes reach the shop, and the delivery pay when the order is handed over.
- The cash is taken **at delivery** and booked to the driver who delivered. The driver page shows the cash in hand and the pay due.
- Each step in the driver app now moves the order: Picked up → Processing → On Going → Delivered.
- Adding a driver asks for NID, address, vehicle, licence and an emergency contact.

| Order: driver jobs and where the money goes | Driver page |
|---|---|
| ![Order](screenshots/shop-05-order-details.jpg) | ![Driver](screenshots/shop-07-driver-page.jpg) |

| Drivers | Add driver |
|---|---|
| ![Drivers](screenshots/shop-06-drivers.jpg) | ![Add driver](screenshots/shop-08-add-driver.jpg) |

### One money rule for every report (SHOP-39, SHOP-40, SHOP-41, AD-60 … AD-63)

Before the fix, the Revenue page, the shop report, the wallets and the dashboard each counted money in a different way, so their numbers never matched. Now they all use one rule:

> **What the shop keeps = what the customer paid − platform commission (a % of the items, not the delivery charge) − driver pay.**
> Orders count on the day they were delivered.

| Shop: Revenue | Shop: Commission owed and paying it |
|---|---|
| ![Revenue](screenshots/shop-09-revenue.jpg) | ![Commission](screenshots/shop-10-commission.jpg) |
| **Admin dashboard: the admin's own income (commission and plans)** | **Admin: which shops owe commission, record a payment** |
| ![Admin dashboard](screenshots/admin-02-dashboard.jpg) | ![Commission admin](screenshots/admin-07-commission.jpg) |

![Shop report](screenshots/admin-05-shop-report.jpg)

### Location (WEB-39 … WEB-41, SHOP-33 … SHOP-36, AD-59 … AD-61)

- The website shows the visitor's address and the stores near them. The admin sets the distance (km) that customers see shops within.
- A shop's location comes from where the shop is now. If the device is far from the saved spot, the page warns the shop.
- Shops draw their delivery area on a map. Google had removed its drawing tool, so I rebuilt the drawing.
- An address can be filled in from the current location.
- The admin's map lists every shop, and shows which shops customers cannot see and why.

| Website: location bar and nearest stores | Address filled from the current location |
|---|---|
| ![Home](screenshots/web-01-home-light.jpg) | ![Address](screenshots/web-08-checkout-address-autofill.jpg) |
| **Shop: delivery area** | **Admin: shops on the map** |
| ![Area](screenshots/shop-11-delivery-area.jpg) | ![Map](screenshots/admin-04-shops-map.jpg) |
| **Shop: "you are 7.65 km from the saved location"** | **Nearest stores** |
| ![Far away](screenshots/shop-19-location-far-away.jpg) | ![Nearest](screenshots/web-04-nearest-stores.jpg) |

### Website: dark mode, phones and pages (WEB-37, WEB-38, AD-47 … AD-56)

| Dark mode | Phone (dark) | Phone |
|---|---|---|
| ![Dark](screenshots/web-02-home-dark.jpg) | ![Phone dark](screenshots/web-10-location-phone-dark.jpg) | ![Phone](screenshots/web-03-home-phone.jpg) |

| FAQ | Contact (messages now reach the admin, AD-57) |
|---|---|
| ![FAQ](screenshots/web-06-faq.jpg) | ![Contact](screenshots/web-07-contact.jpg) |

### Admin panel pages that were rebuilt

| Login | Add shop |
|---|---|
| ![Login](screenshots/admin-01-login.jpg) | ![Add shop](screenshots/admin-09-add-shop.jpg) |
| **Messages from the website's contact form** | **App settings (with the customer distance)** |
| ![Messages](screenshots/admin-10-messages.jpg) | ![Settings](screenshots/admin-08-app-setting.jpg) |

| Admin dashboard (dark) | Shop dashboard | Shop staff and their permissions |
|---|---|---|
| ![Dark dashboard](screenshots/admin-03-dashboard-dark.jpg) | ![Shop dashboard](screenshots/shop-01-dashboard.jpg) | ![Staff](screenshots/shop-13-staff.jpg) |

### More pages

| Admin: customers | Admin: revenue |
|---|---|
| ![Customers](screenshots/admin-11-customers.jpg) | ![Revenue](screenshots/admin-06-revenue.jpg) |
| **Customer: saved addresses** | **All services** |
| ![Addresses](screenshots/web-09-manage-addresses.jpg) | ![Services](screenshots/web-05-all-services.jpg) |
| **Shop profile: location on the map** | **POS sale delivered by a driver** |
| ![Profile](screenshots/shop-12-profile-location.jpg) | ![POS delivery](screenshots/shop-23-pos-deliver-by-driver.jpg) |

---

## Tools

Laravel 10 / PHP / MySQL · Blade, Bootstrap 4, Tailwind · Google Maps API · Stripe, Razorpay, Paystack (test mode) · **Playwright** (end-to-end tests) · **k6** (load test) · headless Chrome over the DevTools protocol for screenshots and page scans · Google Sheets for the bug report · Git.

The test data in the screenshots (shops, customers, emails) is demo data from a local copy of the project.

---

## Every change, in order

Each commit on the `qa` branch (all authored by Arifur Rahman Tuhin, [@tuhin-swe](https://github.com/tuhin-swe)), oldest first. The ID links each change to its row in the bug report.

| # | Date | Commit | Change |
|---|---|---|---|
| 1 | 2026-10-28 | `03f6106a` | fix(WEB-01): link Log In / Register tabs on auth pages |
| 2 | 2026-10-28 | `c0d72144` | fix(WEB-02): add Confirm Password to website registration |
| 3 | 2026-10-28 | `5028c430` | fix(WEB-06): add Back to Home link on website auth pages |
| 4 | 2026-10-28 | `c5e07665` | fix(WEB-04): send vendors to their dashboard, not the website Home |
| 5 | 2026-10-28 | `b77608ef` | fix(WEB-05): sync missing ACL permissions on migrate |
| 6 | 2026-10-28 | `2ebc19bb` | fix(WEB-07): show service photos full width on Home service cards |
| 7 | 2026-10-28 | `f3da1d36` | fix(WEB-08): All Services filter loses services without a slug |
| 8 | 2026-10-28 | `d0fcf896` | fix(WEB-09): smaller, shared page title banner on website pages |
| 9 | 2026-10-28 | `c5a54975` | fix(WEB-09): make the page title banner shorter |
| 10 | 2026-10-28 | `6b29d598` | fix(WEB-10): Book Now does nothing when the location is not ready |
| 11 | 2026-10-28 | `d49e161f` | fix(WEB-11): show clear messages and readable text on the product page |
| 12 | 2026-10-28 | `1649fde0` | fix(WEB-12): customer pages show sign-in instead of a bare 403 |
| 13 | 2026-10-28 | `de3fbfe0` | fix(WEB-13): load checkout time slots and redesign the checkout page |
| 14 | 2026-10-28 | `b9cae611` | fix(WEB-13): shorten the checkout page |
| 15 | 2026-10-28 | `5b189450` | fix(WEB-14): select cart items by default on checkout |
| 16 | 2026-10-28 | `a4ef4dec` | fix(WEB-15): change item quantity in the header cart and on checkout |
| 17 | 2026-10-28 | `d686b636` | fix(WEB-16): Place Order fails silently and keeps old error messages |
| 18 | 2026-10-29 | `8fe297a4` | fix(AD-12): dashboard login with shop credentials opened the website Home |
| 19 | 2026-10-29 | `92d7dfde` | fix(WEB-17): How It Works step numbers looked clickable |
| 20 | 2026-10-29 | `8693228b` | feat(website): eye-catching How It Works photo |
| 21 | 2026-10-29 | `728ee183` | Revert "feat(website): eye-catching How It Works photo" |
| 22 | 2026-10-29 | `2966cabd` | fix(WEB-18): show Terms & Conditions and Privacy Policy as website pages |
| 23 | 2026-10-29 | `9f9b728f` | fix(WEB-18): remove the 'See also' links from the legal pages |
| 24 | 2026-10-29 | `7d167aff` | fix(WEB-19): friendly, centered empty state on Find Stores Near You |
| 25 | 2026-10-29 | `27a3e8b4` | fix(WEB-20): make the 'Ready to Get Started?' strip above the footer smaller |
| 26 | 2026-10-29 | `9af4a143` | fix(WEB-21): selected item in the customer dashboard menu is not visible |
| 27 | 2026-10-29 | `3f179703` | fix(WEB-22): customers can change their profile photo in Settings |
| 28 | 2026-10-29 | `2686f423` | fix(WEB-23): make the delivery address phone number clearly required |
| 29 | 2026-10-29 | `636d71fe` | fix(WEB-24): My Orders and Addresses use the dashboard layout with the side menu |
| 30 | 2026-10-29 | `c16ad01f` | fix(WEB-25): stop calling every customer a 'Premium Member' |
| 31 | 2026-10-29 | `7e7eed73` | feat(WEB-25): show the Customer ID in the customer dashboard |
| 32 | 2026-10-29 | `4efb0dfc` | Revert "feat(WEB-25): show the Customer ID in the customer dashboard" |
| 33 | 2026-10-29 | `8495a2f0` | fix(WEB-26): smoother header cart dropdown |
| 34 | 2026-10-29 | `663be328` | fix(WEB-27): variant tab arrows only show when the tabs overflow |
| 35 | 2026-10-29 | `39914726` | fix(WEB-28): clean favourite (heart) button on store cards |
| 36 | 2026-10-29 | `39086915` | fix(WEB-29): clearer, more useful Favorite Stores list |
| 37 | 2026-10-29 | `67b1416c` | fix(WEB-30): footer 'How It Works' link lands on the How It Works section |
| 38 | 2026-10-29 | `29de690b` | fix(WEB-30): no jump when opening How It Works from the footer |
| 39 | 2026-10-29 | `33824f5b` | fix(SHOP-02): subscription payment shows real gateway logos and a working Proceed button |
| 40 | 2026-10-29 | `d2d16fb4` | fix(SHOP-02): cleaner subscription payment page |
| 41 | 2026-10-29 | `28452f82` | fix(SHOP-03): security fix (access control) |
| 42 | 2026-10-29 | `0473fef9` | fix(SHOP-04): clearer, friendlier subscription plan list |
| 43 | 2026-10-29 | `ec5914a3` | fix(SHOP-05): clearer Services page for shops |
| 44 | 2026-10-29 | `522d1b66` | fix(SHOP-06): a quarterly plan lasts 90 days, not 120 |
| 45 | 2026-10-29 | `d88bac7a` | fix(SHOP-07): explain where Razorpay and Paystack ask for card details |
| 46 | 2026-10-29 | `1bf4b920` | fix(WEB-31): vendor sign-up without a shop name shows an error, not a 500 |
| 47 | 2026-10-29 | `207964fd` | fix(SHOP-08): security fix (access control) |
| 48 | 2026-10-29 | `cd49bda4` | fix(AD-01, AD-03): tidy profile photo upload with allowed types and size |
| 49 | 2026-10-29 | `60840cf0` | fix(AD-04): the notification bell opens a list of the latest orders |
| 50 | 2026-10-29 | `de99f867` | fix(AD-05, AD-06): image preview and upload rules for shop logo, banner and signature |
| 51 | 2026-10-29 | `a0b52dcd` | fix(AD-07): Payment Gateways page is never empty |
| 52 | 2026-10-29 | `773959ec` | fix(AD-09): show the POS receipt when POS Printer is selected |
| 53 | 2026-10-29 | `0e4d6760` | fix(AD-10): Add Driver form takes a photo and date of birth |
| 54 | 2026-10-29 | `ab9cc451` | fix(AD-11): "Select all" checkboxes work on shop services and notifications |
| 55 | 2026-10-29 | `f046d627` | fix(AD-12): Add Shop requires a password and a numeric delivery charge |
| 56 | 2026-10-29 | `23c606b1` | fix(AD-04): bell badge counts unseen orders and clears when opened |
| 57 | 2026-10-29 | `5db911d0` | fix(AD-13): language switcher with country flags |
| 58 | 2026-10-29 | `4de9e928` | fix(SHOP-09): explain an empty Service list on Add Variant and Add Product |
| 59 | 2026-10-29 | `eaa5e400` | fix(AD-14): clear page instead of a bare 403 when the signed-in account has no access |
| 60 | 2026-10-29 | `244beebb` | fix(AD-15): forms start under the header instead of the middle of the screen |
| 61 | 2026-10-29 | `765cf586` | fix(AD-16): dashboard pages no longer jump while loading or switching pages |
| 62 | 2026-10-29 | `0226fef9` | fix(AD-16): pages paint once instead of flashing while they load |
| 63 | 2026-10-29 | `43855f85` | fix(SHOP-09): same empty-service notice on Additional Service forms |
| 64 | 2026-10-29 | `00f1a221` | fix(SHOP-10): shops add the admin's services to their own list |
| 65 | 2026-10-29 | `5087c854` | feat(SHOP-11): shops request new services, the admin approves or rejects them |
| 66 | 2026-10-29 | `741a8e3b` | feat(SHOP-11): a service a shop requested stays private to that shop |
| 67 | 2026-10-29 | `8e3cb419` | feat(AD-08): add, edit, delete and use invoice templates |
| 68 | 2026-10-29 | `8e95531e` | fix(WEB-32): security fix (access control) |
| 69 | 2026-10-29 | `e102e875` | fix(AD-17): invoice previews and PDFs load instead of spinning forever |
| 70 | 2026-10-29 | `9ef9bfc6` | fix(AD-18): saving a setting says whether it was saved |
| 71 | 2026-10-31 | `618a650e` | fix(AD-19): clearer Web Settings page with a section menu |
| 72 | 2026-10-31 | `de336e55` | fix(AD-19): Web Settings sections as tabs instead of a second side menu |
| 73 | 2026-10-31 | `70810044` | fix(AD-19): Web Settings fits phones, tablets and small laptops |
| 74 | 2026-10-31 | `4a5ce7ed` | fix(AD-20): dashboard pages fit phones, tablets and small laptops |
| 75 | 2026-10-31 | `e1c085d6` | fix(AD-21): clearer order details page |
| 76 | 2026-10-31 | `a3e31ddd` | fix(SHOP-12, AD-22): security fix (access control) |
| 77 | 2026-10-31 | `b5f2b988` | fix(SHOP-13): pages a shop has no permission for show "no access", not a 500 |
| 78 | 2026-10-31 | `3971cada` | fix(SHOP-14): each variant of a service gets its own position |
| 79 | 2026-10-31 | `3b892910` | fix(SHOP-15): a shop without a location can set it on the profile map |
| 80 | 2026-10-31 | `74749c5b` | fix(AD-23): one Back button, in one place, on every dashboard page |
| 81 | 2026-10-31 | `32e2ffe9` | fix(SHOP-16): every field the server requires shows a red * on its label |
| 82 | 2026-10-31 | `ed52a284` | fix(SHOP-17): image fields show one thumbnail that swaps when a new file is picked |
| 83 | 2026-10-31 | `a2e035de` | fix(AD-24): every image upload uses the same upload box with the saved image |
| 84 | 2026-10-31 | `e62f1f42` | fix(AD-25): broken layouts found by checking every dashboard page on desktop and phone |
| 85 | 2026-10-31 | `de762ded` | fix(SHOP-18): first-time image uploads are kept instead of deleted |
| 86 | 2026-10-31 | `544f4506` | fix(SHOP-19): variant products page redesigned; product position can be saved |
| 87 | 2026-10-31 | `abc0797a` | fix(SHOP-20): variants list redesigned; "Products" opens the products in place |
| 88 | 2026-10-31 | `2a4bb00c` | feat(SHOP-21): clicking a product on the variants list opens its details |
| 89 | 2026-10-31 | `f2f401a2` | fix(SHOP-22): subscription report shows the current plan, totals and clear statuses |
| 90 | 2026-10-31 | `862e1f5a` | fix(SHOP-23): security fix (access control) |
| 91 | 2026-10-31 | `6a2427bf` | fix(WEB-33): order page (variant services) redesigned with pictures, discounts and a clearer basket |
| 92 | 2026-10-31 | `f9ba2fc4` | fix(WEB-34): order page shows "Order placed" after checkout; smaller banner |
| 93 | 2026-10-31 | `21ee176c` | fix(WEB-35): customer order page redesigned with a clear progress tracker and real phone links |
| 94 | 2026-10-31 | `14984e07` | fix(SHOP-24): orders list redesigned with status tabs, clickable rows and clear payment state |
| 95 | 2026-10-31 | `3e7cae4d` | fix(SHOP-25): dashboard title and breadcrumb shown once |
| 96 | 2026-11-01 | `9eacfb7b` | fix(AD-26): shop list redesigned and responsive; edit/view no longer hidden behind hover |
| 97 | 2026-11-01 | `703b56be` | fix(AD-27): new dashboard for admin and shops: correct figures, charts, recent orders |
| 98 | 2026-11-01 | `03404e38` | fix(AD-27): dashboard polish: needs-attention list, sparklines, next 7 days |
| 99 | 2026-11-01 | `dc54da0c` | fix(AD-28): admin services page redesigned as cards with search and filters |
| 100 | 2026-11-01 | `3d87a48e` | fix(AD-29): admin variants page redesigned as shop cards with search; tested |
| 101 | 2026-11-01 | `555dfe24` | revert(AD-29): put back the previous admin variants page design |
| 102 | 2026-11-01 | `f3986d46` | fix(AD-29): admin variants page as a shop list with a details panel |
| 103 | 2026-11-01 | `2cbee7dc` | fix(AD-30, WEB-36): coupons: orders respect dates and minimum, basket shows the real discount; new coupon pages |
| 104 | 2026-11-01 | `d2888b49` | fix(AD-30): coupons shown as ticket-style cards |
| 105 | 2026-11-01 | `44dfc7f2` | fix(AD-29): admin variants shown as cards in the coupon ticket style |
| 106 | 2026-11-01 | `144e2ddf` | fix(AD-31): Back from a variant's products (or a coupon) returns to the chosen shop |
| 107 | 2026-11-01 | `cc8ecc47` | fix(AD-32): subscription plans shown as pricing cards; new plans appear without reload; 'Onetime' type rejected |
| 108 | 2026-11-01 | `9f920ba9` | fix(AD-33): shop report redesigned with totals, ranking, sort and search; correct product and sales numbers |
| 109 | 2026-11-01 | `e505837e` | fix(AD-34): shop-wise report redesigned; Back returns to the shop report; orders can be opened |
| 110 | 2026-11-01 | `75171b45` | fix(AD-35): app banners shown as cards; 'Hidden' choice in the add popup works; errors reopen the popup |
| 111 | 2026-11-01 | `474f1f32` | fix(AD-35): app banners: phone preview of live banners, switch saves without reload, live preview in add/edit |
| 112 | 2026-11-01 | `1c27e2c8` | fix(AD-36): customers list redesigned with order and spending details; search finds phone numbers and full names |
| 113 | 2026-11-01 | `e934ef1c` | fix(AD-36): customers: quick-view side panel, top customers, CSV export |
| 114 | 2026-11-01 | `fc8b6612` | fix(AD-37): drivers shown as cards with job counts; reactivating a driver works; one page for active and deactivated |
| 115 | 2026-11-01 | `153639b4` | fix(AD-37): drivers: side panel with recent jobs, completion bar, driver app badge, sort |
| 116 | 2026-11-01 | `23f6af3b` | fix(AD-37): drivers: compact list layout and paging for long driver lists |
| 117 | 2026-11-01 | `6ae47450` | fix(AD-36, AD-37): customer and driver quick view opens as a centred popup instead of sliding in from the side |
| 118 | 2026-11-01 | `6d72e3ef` | fix(AD-36, AD-37): clicking a customer or driver opens their page again; no popup, no moving cards |
| 119 | 2026-11-01 | `716f05c9` | fix(AD-36, AD-37): clicking a customer or driver opens their page again; no popup, no moving cards |
| 120 | 2026-11-01 | `74afdbe6` | fix(AD-38): admin pages no longer jump right after opening; layout stays put between sections |
| 121 | 2026-11-01 | `b5e8accb` | fix(AD-37): smaller driver cards; restore the App/No app badge style |
| 122 | 2026-11-01 | `0a875dfb` | fix(AD-39): notifications page redesigned; device filter keeps choices; text kept on errors; shows who gets a push |
| 123 | 2026-11-01 | `5ea1279f` | fix(AD-40): sidebar menu 'Booking' renamed to 'Orders' |
| 124 | 2026-11-01 | `4b9f5f3c` | feat(SHOP-26): shop owners add staff and choose what each staff member may do |
| 125 | 2026-11-01 | `55304166` | feat(SHOP-26): owner picks exactly what each staff member may do |
| 126 | 2026-11-01 | `be295a25` | fix(AD-41): security fix (access control) |
| 127 | 2026-11-01 | `b4706d46` | feat(AD-42): Bangla and Arabic filled in; 'Auto translate empty boxes' in the language editor |
| 128 | 2026-11-01 | `daf4ec17` | feat(AD-43): light and dark mode for the admin and shop panel |
| 129 | 2026-11-01 | `5326e732` | fix(AD-43): dark mode: own logo for dark mode, readable hovered rows and badges, dark scrollbars |
| 130 | 2026-11-01 | `db6f92e2` | fix(AD-43): dark mode: remaining light spots (variant product counts, shop details, wallet, invoice choice) |
| 131 | 2026-11-01 | `b8586dd0` | fix(AD-43): dark mode: readable variant tags and other dark text; light mode keeps exact colours |
| 132 | 2026-11-01 | `a9926ae7` | fix(AD-44): coupons list shows active coupons first, then scheduled, then expired |
| 133 | 2026-11-01 | `75fb89a9` | fix(AD-45): shop report Excel files have headings, all figures and the orders |
| 134 | 2026-11-01 | `073f22e3` | fix(AD-43): dark mode: top customers strip, white text in coloured banners, labels on photos |
| 135 | 2026-11-01 | `983a999e` | feat(AD-46): one clean dropdown style across the panel (light and dark) |
| 136 | 2026-11-01 | `7748aec4` | feat(WEB-37): light / dark mode on the customer website |
| 137 | 2026-11-01 | `072572c7` | fix(AD-47): dark mode: text editors, no-access page and banner switches |
| 138 | 2026-11-01 | `e1dfa6ab` | feat(WEB-38): plain page title instead of the green picture banner |
| 139 | 2026-11-02 | `bf927dbe` | feat(AD-48): clearer dashboard login page |
| 140 | 2026-11-02 | `181da79c` | feat(AD-49): clearer Add / Edit Shop form |
| 141 | 2026-11-02 | `554517a1` | feat(AD-50): revenue page opens on this month, with totals and quick ranges |
| 142 | 2026-11-02 | `f17a7443` | feat(AD-51): tidier driver cards |
| 143 | 2026-11-02 | `5af1e3cb` | feat(AD-52): one clear "Sort" menu on Shop report, Drivers and Customers |
| 144 | 2026-11-02 | `d93b3934` | feat(AD-53): same icon dropdown everywhere in the panel |
| 145 | 2026-11-02 | `f716f015` | feat(AD-54): clearer Privacy policy, Terms, Contact us and About us pages |
| 146 | 2026-11-02 | `1666bba9` | fix(AD-55): FAQ page was empty and could not be filled from the dashboard |
| 147 | 2026-11-02 | `46c773e8` | fix(AD-56): dark mode: popup close buttons, active page number and a few fixed colours |
| 148 | 2026-11-02 | `86c1cc57` | feat(AD-57): Messages page for the website Contact form, and messages in the bell |
| 149 | 2026-11-02 | `5f3e7ca9` | feat(SHOP-27): shop drivers with pay per order and a clear cash account |
| 150 | 2026-11-02 | `cbeb6625` | feat(SHOP-28): online payments in the driver account |
| 151 | 2026-11-02 | `d596d2e3` | feat(SHOP-29): full driver profile when adding a driver, and an edit page |
| 152 | 2026-11-02 | `b8a68edb` | feat(SHOP-30): a POS a shop can really sell with |
| 153 | 2026-11-02 | `f53b9a34` | feat(SHOP-31): POS takes cash or the shop's own payment gateways |
| 154 | 2026-11-02 | `2a88674d` | feat(WEB-39): show the visitor's location on the website |
| 155 | 2026-11-02 | `2940bc5e` | fix(SHOP-32): Profile in the top menu said "page not available" for shops and admins |
| 156 | 2026-11-02 | `ad1a6656` | feat(SHOP-33): shop location from where the shop is now |
| 157 | 2026-11-02 | `8c83d085` | feat(SHOP-34): Delivery area page you can draw on, on one page |
| 158 | 2026-11-02 | `33900a09` | feat(SHOP-35): move the shop pin from the Delivery area page |
| 159 | 2026-11-02 | `e4e5af9f` | fix(SHOP-36): shop address saved with a Google plus code in front |
| 160 | 2026-11-02 | `e9cb2337` | fix(AD-58): security fix (access control) |
| 161 | 2026-11-02 | `4c4007c3` | feat(AD-59): Shops on the map with a list, and shop addresses that match their pins |
| 162 | 2026-11-02 | `93231f16` | feat(AD-60): admin sets how far customers see shops; delivery areas on demand on the map |
| 163 | 2026-11-02 | `d441c77f` | fix(WEB-40): website location bar shows the address, not only the area |
| 164 | 2026-11-02 | `59865c31` | fix(AD-61): one rule for which shops customers see; the map says who is hidden and why |
| 165 | 2026-11-02 | `bc2a89c1` | feat(WEB-41): fill a pickup / delivery address from the current location |
| 166 | 2026-11-02 | `b72da4be` | fix(SHOP-37): cash is collected at delivery, not when a driver is sent for pickup |
| 167 | 2026-11-02 | `3006a26d` | fix(SHOP-38): pickup and delivery drivers: assign error, half pay each, order status follows the driver |
| 168 | 2026-11-02 | `67e4a133` | fix(SHOP-39): driver pickup and delivery pay set by the shop; one money rule for wallets and every report |
| 169 | 2026-11-02 | `aa76c5fe` | fix(SHOP-40): dashboard sales count orders on the day they were delivered, plus POS |
| 170 | 2026-11-02 | `b4a2b86a` | feat(SHOP-41): shops see the commission they owe and pay it, in both business modes |
| 171 | 2026-11-02 | `186dfc2b` | feat(AD-62): the admin dashboard shows the admin's income, not the shops' sales |
| 172 | 2026-11-02 | `4d959cc0` | feat(AD-63): admin dashboard: a card for plan sales and one for commission |
| 173 | 2026-11-05 | `22b81188` | feat(WEB-42): the website in the visitor's language - English, Bangla, Arabic |
| 174 | 2026-11-05 | `ca418f94` | feat(AD-64): Web Settings content in other languages, with auto translate |
| 175 | 2026-11-05 | `31333486` | fix(AD-65): the language editor saves every text, and lists the website's texts |
| 176 | 2026-11-05 | `87c9c998` | feat(SHOP-42): a simpler driver page - two amounts, each with its one action |
| 177 | 2026-11-05 | `1a0a2b12` | fix(WEB-43): button text stays inside the button in Bangla and Arabic |
| 178 | 2026-11-05 | `ab19681e` | fix(WEB-44): forgot password page no longer throws a script error |
| 179 | 2026-11-05 | `d5c94e3d` | fix(WEB-45): My Orders "View Details" shows its eye icon |
| 180 | 2026-11-05 | `a2bb7c76` | fix(SHOP-43): a shop's staff member sees no links to pages they may not open |
| 181 | 2026-11-08 | `4ce2f5db` | fix(AD-66): the admin is told when a shop asks for a new service |
| 182 | 2026-11-08 | `45ab3809` | feat(WEB-46): customers get notifications on the website too |
| 183 | 2026-11-08 | `b47113fb` | feat(WEB-47): a cleaner Contact page |
| 184 | 2026-11-08 | `91c1d6af` | fix(WEB-48): in commission mode customers see the shops again |
| 185 | 2026-11-08 | `359f898c` | feat(SHOP-44): an easier delivery area page |
| 186 | 2026-11-08 | `a68cb617` | feat(SHOP-45): "My location" puts a red pin on the exact spot |
| 187 | 2026-11-08 | `2347d3fa` | fix(SHOP-46): a shop can find where to change its logo and banner |
| 188 | 2026-11-08 | `4f6146f9` | fix(WEB-49): "Use my current location" works when the server cannot reach Google |
| 189 | 2026-11-08 | `284f7bcd` | fix(WEB-50): a placed order is never reported as "Order not placed" |
| 190 | 2026-11-08 | `c33a5f18` | fix(WEB-51): add order_products.price, which the test server was missing |
| 191 | 2026-11-08 | `8d06bd93` | fix(SHOP-47): add the wallet columns of transactions, which the test server was missing |
| 192 | 2026-11-08 | `ead9509e` | feat(SHOP-48): POS for clothes left at the shop: dates, charges, collect or deliver |
| 193 | 2026-11-08 | `753b4096` | feat(SHOP-49): POS takes part of the money now and the rest at handover |
| 194 | 2026-11-08 | `9f41acad` | feat(SHOP-50): POS sales get their own page, not the website order page |
| 195 | 2026-11-08 | `f64ec4b3` | fix(SHOP-51): receipt of a part-paid POS sale showed a negative change |
| 196 | 2026-11-08 | `1e943346` | fix(SHOP-52): a POS sale collected at the counter booked its cash to a driver |
| 197 | 2026-11-08 | `e26d991c` | feat(SHOP-53): pick a driver before Picked up or Delivered |
| 198 | 2026-11-08 | `f7f13f5a` | style(SHOP-54): driver account history amounts in their card's colour |
| 199 | 2026-11-08 | `1657ed2f` | fix(SHOP-55): assigning a driver no longer marks the order Picked up / Delivered |
| 200 | 2026-11-08 | `7429131b` | style(WEB-52): less empty space above the items on a service page |
| 201 | 2026-11-08 | `c704db6f` | feat(SHOP-56): drivers ask the shop for their pay; payments keep method and TrxID |
| 202 | 2026-11-08 | `0f02d62c` | fix(AD-67): security fix (access control) |
| 203 | 2026-11-08 | `16641695` | feat(SHOP-57): pickup and delivery times on one simple weekly form |
| 204 | 2026-11-09 | `24b89064` | fix(SHOP-58): order status moves one step forward only |
| 205 | 2026-11-09 | `03155b27` | feat(AD-68): driver payout methods are set by the admin, not fixed in code |
| 206 | 2026-11-09 | `115fd7a2` | style(AD-68): clearer driver payout methods page |
| 207 | 2026-11-09 | `113ee091` | fix(AD-69): the panel shows English in Bangla and Arabic |
| 208 | 2026-11-09 | `65604b81` | fix(WEB-53): website headings stay English on the test server in Bangla / Arabic |
| 209 | 2026-11-09 | `5a2bea99` | fix(AD-70): dashboard links sent guests to the website home, not the sign-in |
| 210 | 2026-11-09 | `82fc7e82` | fix(WEB-54): security fix (access control) |
| 211 | 2026-11-09 | `a48ca829` | fix(AD-71): script error on Settings > Social link |
| 212 | 2026-11-09 | `71fc6cea` | fix(SHOP-59): POS online payment page crashed and was slow |
