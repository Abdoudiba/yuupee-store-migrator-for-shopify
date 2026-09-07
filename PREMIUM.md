# Store Migrator — Premium add-on spec

Reference for building the paid tier of **Yuupee Store Migrator for Shopify**.
Companion to `README.md` (which has the free-core architecture). Comments and
generated docs in French per house style; this internal spec is in English to
match `README.md`.

**Status:** M5 — skeleton. Decisions locked 2026-09-06.

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
| `includes/class-stwmp-wizard.php` | hooks the free wizard's seams: adds "API" as a source on Connect, injects a "Choose data" step, renders per-entity pre-flight |
| `includes/importers/class-stwmp-collections.php` | `collection` batch → product categories, nested, membership from `collects` / smart-rule expansion best-effort |
| `includes/importers/class-stwmp-customers.php` | `customer` batch → WC customers + addresses; no password (send reset mail option) |
| `includes/importers/class-stwmp-orders.php` | `order` batch → `WC_Order` via CRUD; line items resolved through the ID map; status mapping (§5); taxes, shipping lines, discount lines, notes |
| `includes/importers/class-stwmp-coupons.php` | `price_rule` + `discount_code` → WC coupons |
| `includes/class-stwmp-preflight.php` | API-side dry run: counts per resource, token scope check, currency mismatch, gift-card / bundle warnings |

---

## 3. Free-core changes required (ship first, as free plugin v1.1.0)

The add-on cannot exist until the WP.org plugin exposes seams. All additive,
all no-ops when the add-on is absent, all review-safe (no premium logic, no
external calls).

1. **`STWM_Run`**
   - already carries `source` (`'csv'`) and `entities` (`['product']`) — formalise:
     `source` ∈ `{csv, api}`, `entities` is an ordered list.
   - add `meta` blob for adapter state (API cursor, shop domain hash) — never the token.

2. **`STWM_Admin` — filterable wizard**
   - `steps()` → `apply_filters( 'stwm_wizard_steps', $steps )` so the add-on can
     splice `choose` between `connect` and `analyze`.
   - `render()` step dispatch → if no local `step_{slug}()` method, fire
     `do_action( "stwm_wizard_render_step_{$slug}", $run )`.
   - `step_connect()` → wrap the CSV form in
     `do_action( 'stwm_connect_before_form', $run )` /
     `apply_filters( 'stwm_connect_sources', [ 'csv' => … ] )`; when >1 source,
     render a source picker.
   - `handle_post()` → `do_action( "stwm_wizard_handle_{$step}", $step )` for
     unknown steps, and `apply_filters( 'stwm_wizard_next_step', $next, $step, $run )`.

3. **`STWM_Admin::render_preflight()`** → after the core list,
   `do_action( 'stwm_after_preflight', $run )` so per-entity checks render inline.

4. **`STWM_Queue::handle_batch()`** already ends with
   `do_action( 'stwm_process_batch_dispatch', $payload )` — keep. Add the same
   dispatch for a `finalize` phase: `do_action( 'stwm_run_finalize', $run_id )`
   after the last entity, for the add-on to write 301s / send reset mails.

5. **`STWM_Migration_Map`** — add `get_target( $entity_type, $source_id )` if not
   already public (orders resolve products/customers through it); add
   `source_ids_for_type( $run_id, $type )` for incremental.

6. **Upsell surface (free plugin)** — a "Premium" tab on the Shopify Import
   screen: static feature list + a single outbound link to the sales page.
   No tracking, no phone-home. WP.org allows one clearly-labelled upsell link.

7. **`stwm_premium_active`** filter the add-on sets `true`; free core uses it
   only to hide the upsell tab.

Free plugin bumps to **1.1.0**, changelog "Adds integration hooks for the
premium add-on; no behaviour change for the free importer." Re-run Plugin Check,
`phpcs`, regenerate `.pot`, tag on SVN.

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
- **M5.3** `STWMP_API` client + connection test + `STWMP_Preflight` counts.
  Test against a Shopify development store (free partner account).
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
  `https://<yuupee-site>/sm-premium/update.json`; the JSON is regenerated on
  each release. License key passed as a query arg so we can 403 unlicensed
  update pulls (soft — the grace logic still applies client-side).
- Semver. Free-core minimum version recorded in the add-on header
  (`Requires Plugins` can't express our free plugin's *version*, so the
  bootstrap checks `STWM_VERSION` and shows a notice).

## 8. Open items

- Yuupee sales/landing page for the add-on (one page + LS buy button + docs
  link). Blocked on: which domain / does it live on www.yuupee.com or a
  subpath.
- Shopify API version pin: start `2024-10`, bump quarterly; the client sends a
  fixed version string so a Shopify deprecation can't silently change shapes.
- Partner/development Shopify store for testing (free) — create under the Yuupee
  Partner account.
- Decide refund granularity for `partially_refunded` (line-level vs a single
  adjustment line) — spike during M7.
