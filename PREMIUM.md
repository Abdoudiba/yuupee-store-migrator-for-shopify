# Store Migrator — Premium add-on spec

Reference for building the paid tier of **Yuupee Store Migrator for Shopify**.
Companion to `README.md` (which has the free-core architecture). Comments and
generated docs in French per house style; this internal spec is in English to
match `README.md`.

**Status:** M5.1 ✅ (free-core hooks shipped in v1.1.0) · M5.2 ✅ (add-on
skeleton + Lemon Squeezy license client) · M5.3 ✅ (`STWMP_API` read-only
Admin API client + connection test wired into the Premium screen +
`STWMP_Preflight` counts/scopes/currency) · M5.4 ✅ (`STWMP_Wizard`: additive
"import also from the Admin API" panel on Connect, inserted "Choisir les
données" step, per-entity pre-flight block on Analyze — proven end to end in
WP Playground; free core carries the new `stwm_wizard_handle_*` +
`stwm_wizard_next_step` seams, unreleased on `1.2.0-dev`). Next: M6 Collections
importer. Decisions locked 2026-09-06.

---

## 1. Decisions (locked)

| Question | Decision | Why |
|----------|----------|-----|
| Checkout + licensing | **Lemon Squeezy** | Merchant of Record (LS handles global VAT/sales-tax), hosted checkout, first-class License API (`activate` / `validate` / `deactivate`). No store infra to run. ~5 % + 50 ¢ per sale is worth not being liable for tax in 40 countries. Freemius explicitly rejected (revenue share). |
| Packaging | **Separate add-on plugin** `yuupee-store-migrator-shopify-premium` | Own repo, own zip, distributed from our site only — never WP.org. Requires the free plugin ≥ the version that ships the hook seams. Clean review story: the WP.org plugin has zero premium code, only `apply_filters` / `do_action` seams. |
| v1.0 feature cut | **Admin API connect + Collections → categories + Customers + Orders (status map) + Discount codes → coupons** | The "bring my sales history and customers" bundle — that is what a store owner pays to not redo by hand. |
| v1.1 | 301-redirect export, blog posts + pages | |
| v1.2 | Incremental re-runs ("orders since last run") | Reads the ID map; needs the API cursor stored per run. |
| Price (launch) | **$79 single-site / $119 3-site / $249 agency (5)**, 1 year of updates + support, non-expiring license (keeps working, updates stop) | Slightly above the README's old $69/99/199 — LS fees + we include a real support promise. |
| Non-goals | hosted SaaS, per-migration billing, multi-cart, matching Cart2Cart breadth | unchanged from `README.md` |

---

## 2. Repo layout

```
~/projects/store-migrator-for-woocommerce/     free core — WP.org (slug yuupee-store-migrator-for-shopify)
~/projects/ysm-svn/                             its WP.org SVN working copy
~/projects/yuupee-store-migrator-shopify-premium/   THIS add-on — private repo, sold via Lemon Squeezy
```

The add-on's plugin slug / main file: `yuupee-store-migrator-shopify-premium/yuupee-store-migrator-shopify-premium.php`.
Code prefix: **`stwmp_` / `STWMP_`** (free core stays `stwm_` / `STWM_`).
Text domain: `yuupee-store-migrator-shopify-premium` (its own `languages/`).

### Add-on file map (target)

| File | Role |
|------|------|
| `yuupee-store-migrator-shopify-premium.php` | bootstrap; checks free core is active + version-compatible; constants; loads the rest on `plugins_loaded` priority 20 (after free core) |
| `includes/class-stwmp-license.php` | Lemon Squeezy License API client — activate / validate / deactivate, transient-cached status, weekly re-validate via Action Scheduler, admin notice when invalid |
| `includes/class-stwmp-updater.php` | self-hosted update check (LS "Get latest version" / a small `update.json` on our site) — plugin-update-checker library (YahnisElsts), MIT |
| `includes/class-stwmp-settings.php` | "Shopify Import → Premium" settings: license key field, Shopify store domain + Admin API token, connection test |
| `includes/class-stwmp-api.php` | Shopify Admin API REST client: base URL from store domain, `X-Shopify-Access-Token`, 2 req/s leaky-bucket honouring `Retry-After` + the `X-Shopify-Shop-Api-Call-Limit` header, cursor pagination (`Link: rel="next"`), typed errors |
| `includes/class-stwmp-wizard.php` | hooks the free wizard's seams: an "import also from the Admin API" panel on Connect (additive to the CSV upload), an inserted "Choisir les données" step, and the per-entity pre-flight block on Analyze |
| `includes/importers/class-stwmp-collections.php` | `collection` batch → product categories, nested, membership from `collects` / smart-rule expansion best-effort |
| `includes/importers/class-stwmp-customers.php` | `customer` batch → WC customers + addresses; no password (send reset mail option) |
| `includes/importers/class-stwmp-orders.php` | `order` batch → `WC_Order` via CRUD; line items resolved through the ID map; status mapping (§5); taxes, shipping lines, discount lines, notes |
| `includes/importers/class-stwmp-coupons.php` | `price_rule` + `discount_code` → WC coupons |
| `includes/class-stwmp-preflight.php` | API-side dry run: counts per resource, token scope check, currency mismatch, gift-card / bundle warnings |

---

## 3. Free-core changes

### 3a. Shipped in v1.1.0 (M5.1 ✅)

All additive, all no-ops when the add-on is absent, all review-safe (no premium
logic, no external calls). In `includes/class-stwm-admin.php`:

- `steps()` → `apply_filters( 'stwm_wizard_steps', $steps )` (falls back to the
  built-in list if a filter returns something empty/non-array). Splices in extra
  steps; `current_step()` / `render_steps_nav()` / `handle_post()` all read
  through `steps()` so filter-added slugs are already valid and numbered.
- `render()` → for a step with no `step_{slug}()` method, fires
  `do_action( "stwm_wizard_render_step_{$slug}", STWM_Run::current() )`.
- `step_connect()` → `do_action( 'stwm_connect_before_form' )` and
  `do_action( 'stwm_connect_after_form' )` around the CSV upload form.
- `step_analyze()` → `do_action( 'stwm_after_preflight', $run )` right after the
  core pre-flight list.

`STWM_Queue::handle_batch()` already ends with
`do_action( 'stwm_process_batch_dispatch', $payload )` — unchanged, that is how
the premium entity processors (collection/customer/order/coupon) get dispatched.

### 3b1. Added for M5.4 (in free core `1.2.0-dev`, not yet released)

In `includes/class-stwm-admin.php::handle_post()`, both additive and no-op
without a listener:

- `do_action( "stwm_wizard_handle_{$step}", STWM_Run::current() )` in the
  `switch`'s `default:` — processes the POST of a filter-added step (nonce
  `stwm_wizard_{slug}` already checked by the core).
- `apply_filters( 'stwm_wizard_next_step', $next, $step, $run_id )` after the
  switch — lets an add-on redirect elsewhere (used to route Connect →
  `choose-data` when an API import is pending). Unknown slugs fall back to
  `report`.

The add-on's `STWMP_Wizard` consumes these plus the existing `stwm_wizard_steps`
/ `stwm_wizard_render_step_*` / `stwm_connect_after_form` / `stwm_after_preflight`.
The **additive-panel** design (chosen over a source picker) means
`stwm_connect_sources` was **not** needed. Ship `1.2.0` to WP.org before the
premium add-on's public release (M8) — the add-on's `STWMP_MIN_CORE_VERSION` is
`1.2.0-dev` during joint dev, bump to `1.2.0` at premium release.

### 3b. Still deferred to when the consuming code exists (M6+)

- `do_action( 'stwm_run_finalize', $run_id )` after the last entity — for 301s /
  password-reset mails (M9).
- `STWM_Migration_Map::get_target()` public + `source_ids_for_type()` — orders
  (M7) and incremental (M10).
- `STWM_Run`: formalise `source ∈ {csv, api}`, add a `meta` blob for adapter
  state (API cursor, shop-domain hash — never the token).
- "Premium" upsell tab on the Shopify Import screen (static feature list + one
  outbound link to `test.yuupee.com/store-migrator-premium`, no phone-home),
  hidden when `apply_filters( 'stwm_premium_active', false )` is true. Ship with
  a later free bump so the tab and the product launch land together.

---

## 4. Lemon Squeezy licensing

LS License API (no auth header needed for these three; the key *is* the secret):

| Action | Endpoint | When |
|--------|----------|------|
| Activate | `POST https://api.lemonsqueezy.com/v1/licenses/activate` `{ license_key, instance_name }` | user pastes key + clicks Activate. `instance_name` = `home_url()`. Store `instance.id` + `license_key`. |
| Validate | `POST /v1/licenses/validate` `{ license_key, instance_id }` | on activate, then weekly via `as_schedule_recurring_action`, and before starting an API migration. |
| Deactivate | `POST /v1/licenses/deactivate` `{ license_key, instance_id }` | on "Deactivate" button and in the add-on's `register_deactivation_hook` (best-effort) so the seat frees. |

- Cache the last good status in a transient (`stwmp_license`, 12 h) **and** a
  non-expiring option as fallback, so a temporary LS outage or a blocked
  outbound request doesn't strand a paying user mid-migration. Grace: allow
  runs while `status === 'active'` OR (last good check < 14 days ago).
- `status` values handled: `active`, `inactive`, `expired`, `disabled`.
  `expired` → allow current version to keep working, block updates, show notice.
- Never block the *free* importer. The add-on gates only its own entity types:
  if the license is invalid, `stwmp_*` importers refuse to enqueue and the
  wizard's API source is hidden with an inline "activate your license" note.
- Store domain + Admin API token: `update_option` autoload **no**; token shown
  masked after save; offer "test connection" (calls `GET /admin/api/2024-10/shop.json`).
  Document that the token is entered by the merchant (no phone-home) — same rule
  as the free plugin.

Product setup in LS: one product "Yuupee Store Migrator for Shopify — Premium",
three variants (1 / 3 / 5 sites) → `activation_limit` 1 / 3 / 5. 12-month
subscription with "license stays valid after cancel" = off renewals still allowed.

---

## 5. Shopify order status → WooCommerce

| Shopify `financial_status` | Shopify `fulfillment_status` | WooCommerce status |
|---|---|---|
| paid | fulfilled | `completed` |
| paid | partial / null | `processing` |
| pending / authorized | any | `on-hold` |
| refunded | any | `refunded` |
| partially_refunded | any | `processing` (+ a refund line + order note) |
| voided | any | `cancelled` |
| any + `cancelled_at` set | — | `cancelled` |

- `paid_at` → `date_paid`; `closed_at` noted; `processed_at` → `date_created`.
- Line items: resolve `product_id` / `variant_id` through the ID map; if the
  product wasn't imported (deleted in Shopify), keep the line as a
  name+price+qty item with no product link and log a warning — never fail the
  order.
- Money: import totals verbatim from Shopify (`total_price`, `total_tax`,
  `total_shipping_price_set`, discount allocations); do **not** recompute — a
  historical order must match the customer's receipt.
- Transactions/gateways: store `payment_method_title` = Shopify gateway name;
  no real payment token.
- `customer` → ID map; guest orders keep billing/shipping only.

---

## 6. Build order (small, tested steps)

- **M5.1** free-core seams → free plugin 1.1.0 (this is the gating step). Test:
  free importer still green end-to-end in Playground; new hooks fire with no
  listener attached and change nothing.
- **M5.2** add-on skeleton: bootstrap + version guard + `STWMP_License` (LS
  client, settings field, activate/validate/deactivate, weekly cron, notices).
  Test against a real LS test-mode product + key.
- **M5.3** ✅ `STWMP_API` client (read-only GET, ~2 req/s leaky bucket +
  `Retry-After` + `X-Shopify-Shop-Api-Call-Limit`, cursor pagination via `Link`
  `rel="next"`, typed `WP_Error`s) + connection test (`GET shop.json`, button on
  the Premium screen) + `STWMP_Preflight` (per-resource counts, `access_scopes`
  check with `write_*` satisfying `read_*`, Shopify↔WC currency mismatch,
  gift-card/bundle + smart-collection warnings). Still to verify against a real
  Shopify development store (free partner account).
- **M5.4** wizard: API source on Connect + "Choose data" step + per-entity
  pre-flight rendering.
- **M6** Collections importer (simplest graph) end-to-end.
- **M7** Customers, then Orders (the hard one — status map, money, line-item
  ID-map resolution, refunds).
- **M8** Coupons. → **Premium v1.0 release** (LS live, sales page, changelog).
- **M9 / v1.1** 301 redirects (+ a Redirection-plugin export and an `.htaccess`
  / nginx snippet), blog posts + pages.
- **M10 / v1.2** incremental mode.

## 7. Distribution / release

- Add-on zip built by `bin/build.sh` (strips `.git`, `vendor` dev deps, tests),
  version-stamped, uploaded to LS as the downloadable + mirrored on our site
  for the updater's `update.json`.
- `plugin-update-checker` (YahnisElsts, MIT) points at
  `https://test.yuupee.com/sm-premium/update.json`; the JSON is regenerated on
  each release. License key passed as a query arg so we can 403 unlicensed
  update pulls (soft — the grace logic still applies client-side).
- Semver. Free-core minimum version recorded in the add-on header
  (`Requires Plugins` can't express our free plugin's *version*, so the
  bootstrap checks `STWM_VERSION` and shows a notice).

## 8. Open items

- Sales/landing page for the add-on lives at **`https://test.yuupee.com/store-migrator-premium`**
  (decided 2026-09-06) — one page + LS buy button + docs link. The updater's
  `update.json` is served from `https://test.yuupee.com/sm-premium/update.json`.
  Still to build the page itself and stand up the update endpoint.
- Shopify API version pin: start `2024-10`, bump quarterly; the client sends a
  fixed version string so a Shopify deprecation can't silently change shapes.
- Partner/development Shopify store for testing (free) — create under the Yuupee
  Partner account.
- Decide refund granularity for `partially_refunded` (line-level vs a single
  adjustment line) — spike during M7.
