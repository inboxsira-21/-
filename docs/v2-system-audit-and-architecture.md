# V2: System Audit and Architecture

Date: 2026-11-18
Repository: inboxsira-21/-
Working branch: rebuild/v2-design

## Executive decision

Build V2 as a clean, separate application rather than extending the current single-file application. Keep V1 intact while V2 is built and tested. Do not delete or overwrite V1 data or code. Switch the public site only after acceptance tests and data migration have passed.

Primary non-functional requirements, in order:
1. Correct stock and transaction integrity
2. Fast interaction on low- to mid-range phones
3. Reliable operation across phones, tablets, and desktop browsers
4. Small frontend payload and low maintenance burden
5. Safe growth to more products, users, locations, and sales integrations

## 1. Repository and V1 audit

### What was actually inspected
The default branch currently contains:
- `index.html` (about 34 KB in GitHub's reported file size; a single-page HTML/CSS/JavaScript application)
- `README.md`
- `สต๊อก 2_36 อิ้ง.md`
- `docs/qr-inventory-plan.md`

The separate `inboxsira-21/stock` repository currently contains only a README in its default-branch tree. It is not a populated replacement application.

The current HTML app:
- Implements camera scanning using browser BarcodeDetector when available, with jsQR as fallback, and image-upload scanning
- Accepts pasted QR text and scanner keyboard input
- Has receive and issue modes
- Interprets a hard-coded tracking-label format and hard-coded product/source mappings
- Stores events in browser localStorage
- Optionally synchronizes events to a Google Apps Script URL configured by the user
- Displays aggregate counts and the latest 15 activity events
- Exports CSV
- Has a local setting for a display name and an option to allow issue without a recorded receipt

### Confirmed V1 weaknesses from source inspection
1. **Device-local storage is the default.** Without a configured remote sync endpoint, data belongs to that browser/device. Different phones and computers do not automatically share one authoritative stock balance.
2. **The data model is label-count based.** Stock is calculated as a count of unique tracking IDs received minus those issued. It does not represent arbitrary decimal quantities such as 2.5 kg, units, packs, or unit conversions.
3. **One receive and one issue per tracking ID.** The current record reconstruction keeps the first receive and first issue event for a tracking ID. It is not a general stock movement ledger for repeated partial movements.
4. **Product identity and QR parsing are coupled to hard-coded rules.** Product/source abbreviations and label structure are embedded in JavaScript, so adding products or changing labels requires code changes.
5. **No product/package master-data screens are present in the inspected HTML.** There is no full product catalogue, reusable packaging definition, unique package-instance registry, or QR label management workflow.
6. **No real user authentication or server-enforced role checks are visible in the inspected frontend.** The name field is an attribution label, not proof of identity. Client-side visibility or flags cannot secure inventory actions.
7. **No authoritative transactional stock operation is visible in the frontend.** Two devices can act on stale local copies. Stock validation and write operations need to be atomic on the server/database.
8. **Sync correctness depends on an external Google Apps Script endpoint that is not present in this repository tree.** Its authentication, locking, conflict handling, quotas, backups, and error recovery cannot be audited from this repo. These remain unknown until the script and its deployment configuration are inspected.
9. **The shared link embeds the sync URL and key in the URL fragment.** Fragments are not sent as normal HTTP request paths, but the secret is still exposed to anyone who receives or can inspect that link. It is unsuitable as a user identity or role system.
10. **The app is a single large HTML file.** This is easy to host, but mixing UI, scanning, parsing, business rules, storage, sync, and reporting makes changes and testing harder.
11. **External scanner library is loaded from a CDN.** If that CDN is unavailable or blocked, fallback scanning may fail. The app should package the scanner dependency locally or provide a clearly tested fallback.
12. **The current UI's totals are counts of labels (the page says ใบ), not general quantity balances.** This is a semantic mismatch for a stock system that needs kg, pieces, boxes, and other units.

### Important limits of this audit
- The Google Apps Script source, spreadsheet, deployment settings, and any data outside this repository were not available in the inspected tree. Their actual behavior is not assumed.
- The old label-generation source is not in the current default-branch tree inspected here.
- This is a source-level audit, not a load test or a live production-data audit.
- Do not claim that all historical data is in this repository. Browser localStorage and external Sheets may contain additional records.

## 2. V2 product scope

### Core modules
1. Dashboard: current stock, low-stock alerts, today's movements, quick actions
2. Scan: camera, image scan if practical, USB/Bluetooth keyboard scanner, and manual code entry
3. Receive: scan product QR and enter quantity; or scan package QR and confirm its preset quantity
4. Issue: select/scan product, enter quantity, choose reason/reference, validate available stock
5. Products: create/edit/archive/search, SKU, name, category, base unit, optional barcode, reorder threshold
6. Packages: package templates and individually identified package labels
7. Stock: balances by product and optionally by location/lot
8. Movement history: searchable, filterable, exportable audit trail
9. Reports: daily/monthly received, issued, adjustments, stock valuation when cost data is enabled
10. Users and permissions: owner, manager, staff, read-only
11. Settings and backups: units, locations, numbering, data export and restore procedure

### QR semantics
- PRODUCT QR: stable identifier for a product. Scanning opens a fast quantity-entry flow. It does not itself imply a quantity.
- PACKAGE QR: a unique package-instance identifier linked to a product and a fixed quantity/unit. Scanning shows product and preset quantity and requires confirmation before posting.
- Package templates (e.g. 1 crate = 20 kg) are separate from unique package instances. A template can generate many individually unique QR labels.
- Each unique package-instance QR is single-use by default. A second scan must show its previous transaction and must not add stock again. An authorized reversal/reopen flow must be explicit and audited.
- If the business later needs reusable containers, model container return/deposit separately; do not silently make a single-use package QR reusable.

## 3. Recommended architecture

### Frontend
- Keep GitHub Pages as the static web host initially.
- Build a small responsive Progressive Web App (PWA).
- Prefer TypeScript with a lightweight build tool (Vite) and modular source files; do not introduce a large UI framework unless a measured need appears.
- Keep the production bundle small. Lazy-load reports and admin-only screens.
- Cache static application assets with a service worker. Never cache sensitive API responses indiscriminately.
- Keep scan flow short: scan -> identify -> enter/confirm quantity -> save -> immediate clear feedback.
- Support touch, keyboard, camera, and external USB/Bluetooth scanners.

### Backend and database
Recommended default: Supabase (managed PostgreSQL + Auth + Row Level Security + database functions), subject to the owner creating a project and providing its public project URL and anon key.
- GitHub Pages contains only public frontend configuration. Never put service-role keys, database passwords, or other server secrets in the frontend.
- Enforce permissions with database Row Level Security and server-side functions, not hidden buttons.
- Use PostgreSQL transactions for receive/issue/adjustment. A stock check and stock movement must commit atomically.
- Use database unique constraints for idempotency and one-time package redemption.
- Use database indexes on product_id, created_at, location_id, transaction type, and package_instance_id.
- Do not use Google Sheets as the authoritative stock ledger for concurrent production use. It may remain as an export/report destination.
- Realtime subscriptions are optional; first make normal reads/writes correct and fast. Add realtime only where it improves actual workflows.

### Offline behavior
- Cache the app shell for opening the app when offline.
- Allow offline scanning and draft transaction capture in a local queue with a generated idempotency key.
- Clearly mark offline entries as pending; do not claim they are committed to shared stock until server acknowledgement.
- On reconnect, submit queued entries idempotently. Server validates current stock and permissions. Conflicts must be shown for user resolution; never silently overwrite.
- Do not promise fully concurrent offline stock accuracy. Two disconnected devices cannot know each other's latest stock changes.

## 4. Proposed data model

Use UUID primary keys and timestamps in UTC. Display local Thai date/time in the UI. Store quantities as DECIMAL/NUMERIC, never floating-point binary values.

- `organizations`: id, name, settings
- `users/profiles`: auth_user_id, display_name, role, active
- `locations`: id, code, name, active
- `units`: id, code, name, decimal_precision
- `products`: id, sku, name, category_id, base_unit_id, active, reorder_level
- `unit_conversions`: product_id or unit pair, from_unit, to_unit, factor; define permitted conversions explicitly
- `package_templates`: id, product_id, label, quantity_in_base_unit, display_unit, active
- `package_instances`: id, package_template_id, qr_token, status, created_at, redeemed_transaction_id
- `stock_movements`: id, idempotency_key, type, product_id, location_id, quantity_base_unit, unit_cost optional, lot_id optional, package_instance_id optional, reference, note, actor_id, created_at, reversal_of_id optional
- `stock_balances`: product_id, location_id, quantity_base_unit, updated_at; updated in the same database transaction as movement posting
- `lots` (optional at launch, schema-ready): id, product_id, lot_code, received_at, expiry_at, supplier_id
- `audit_logs`: actor_id, action, entity_type, entity_id, before/after metadata, created_at

### Ledger rules
- Every stock change is a movement: receipt, issue, adjustment, transfer-out, transfer-in, or reversal.
- Do not hard-delete posted movements. Correct mistakes with an audited reversal and replacement movement.
- Derive and reconcile balances from the ledger. A materialized/current balance table is maintained transactionally for fast reads.
- Use a single database function for posting each movement, with validation, permission checks, idempotency, and balance update in one transaction.
- For issue: reject if available stock is insufficient, unless an explicit owner-level negative-stock policy is enabled.
- For package redemption: unique constraint on package instance redemption and transaction idempotency prevents double receive.
- Record actor identity from authenticated session, not from a user-entered name field.

## 5. Performance and reliability targets

These are design targets to verify in testing, not measured current performance:
- Initial mobile app shell: target <= 150 KB compressed excluding optional scanner engine; measure and tune.
- Common dashboard read: target p95 <= 500 ms on a normal connection.
- Online stock movement post: target p95 <= 1 second on a normal connection, excluding poor network latency.
- Scanner screen opens quickly and does not load reporting/admin code.
- No full-table download for dashboard or search. Use pagination, indexed filters, and server-side aggregates.
- Debounce search, but never debounce away distinct stock transactions.
- Avoid polling every few seconds. Refresh on demand or use realtime only for visible stock changes.
- Database backups and restore drills are required before production use.
- Add structured error logs and a visible sync status for queued/offline transactions.

## 6. Future growth designed in, not overbuilt at launch

Design the data model so it can support:
- Multiple staff and concurrent devices
- Multiple stores/warehouses and stock transfers
- More products, categories, suppliers, and units
- Kg/g, litre/ml, pieces, bundles, crates, and product-specific conversions
- Purchase cost and stock valuation
- Sales integration and automatic issue on completed sale
- Lot/batch and expiry tracking, with FEFO picking if required
- Reorder alerts and purchasing suggestions
- Import/export and management reports
- More granular roles and approval for adjustments

Do not build all future screens at once. Make the core schema and transaction rules extensible, then ship modules in measured phases.

## 7. Migration and rollout

1. Keep V1 on main branch and preserve its history.
2. Build V2 in `rebuild/v2-design` until tested.
3. Inspect the actual Apps Script and spreadsheet before migration; export browser-local V1 data from every device that contains unique records.
4. Produce a migration preview with counts, duplicate tracking IDs, unknown products, and unparseable labels.
5. Map old label-count records to the new product/unit model only where mapping is reliable. Do not invent quantities or silently discard ambiguous rows.
6. Run V2 in parallel with V1 for a controlled test period, without double-posting live transactions.
7. Reconcile opening stock and movement totals with the owner.
8. Switch GitHub Pages only after acceptance tests pass; keep a rollback path to V1.

## 8. Acceptance tests before launch

- Product QR always opens the correct product and asks for quantity.
- Package QR posts its configured quantity exactly once.
- Repeated package scan never creates a duplicate receipt.
- Concurrent issue attempts cannot drive stock negative unless policy explicitly allows it.
- Decimal quantities and unit conversions round correctly.
- Same idempotency key submitted twice creates one movement.
- Wrong-role user cannot post, edit, or reverse transactions through direct API calls.
- Every movement records authenticated actor, timestamp, quantity, unit, and reference.
- Offline queue visibly distinguishes pending, accepted, and rejected/conflicted entries.
- Data export and restore are verified.
- Camera scanning works on supported mobile browsers; keyboard scanner/manual input remain available.
- Layout and common workflows work on mobile, tablet, and desktop.

## 9. Decisions requiring owner input before production connection

Recommended defaults:
- Database: Supabase PostgreSQL
- Hosting: existing GitHub Pages
- Roles: Owner / Manager / Staff / Read-only
- Stock policy: do not allow negative stock by default
- Package QR: unique single-use instance
- Quantities: decimal, stored in product base unit
- Corrections: reversal + replacement, never hard-delete posted movement
- Offline: queue drafts, server confirms final stock

Before going live, confirm the base units and any required cost/sales workflows. The database design can accommodate them without compromising the lightweight scan-first interface.
