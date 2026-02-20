---
name: salable
description: Monetize apps on beta.salable.app 2.0 using Salable MCP tools and REST API. Use when creating or modifying products, plans, line items, prices, currency options, tiers, and entitlements, including packaging decisions like flat-rate, per-seat, and metered pricing. When asked to monetize or monetise an app, assume scope includes in-app paywall pricing tables, feature gating with entitlements, and subscription management views; determine entitlement mapping with the user before provisioning.
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, mcp__salable__*
---

# Salable Pricing Model Builder

Design and implement Salable 2.0 monetization. Use MCP tools for catalog provisioning writes; use REST API for app runtime surfaces (pricing pages, entitlement checks, subscription management).

## Codebase Context Rules

- Work exclusively with files on disk. Do NOT use git log, git diff, or commit history to understand the project.
- Read actual source files for stack, structure, auth, and existing integrations.

## Salable MCP Prerequisite

Before any provisioning work, check whether Salable MCP tools are available (e.g. `mcp__salable__products_list`).

**If available:** You MUST use MCP for all catalog writes. Do not fall back to REST for provisioning.

**If not available:** Stop and instruct the user:
1. Export API key: `export SALABLE_API_KEY="your_key"` (add to `~/.zshrc` for persistence).
2. The plugin already configures the MCP server. If it's not connecting, check that `SALABLE_API_KEY` is set in your environment.
3. Restart Claude Code and re-run `/salable:salable`.

Do not proceed without a working MCP connection.

## OpenAPI Spec

The `openapi.yaml` is truncated in browsers/WebFetch but the full spec is available via curl. At session start, download it:

```bash
curl -s https://beta.salable.app/openapi.yaml > /tmp/salable-openapi.yaml
```

Always grep this file for exact request/response schemas before writing integration code — do not guess field names or casing. Use `beta.salable.app` contracts only, not `docs.salable.app`.

## Default Monetize Scope

When asked to "monetize/monetise this app", deliver three things:
1. Public (unauthenticated) in-app paywall/pricing table.
2. In-app feature gating with entitlements.
3. In-app subscription management views (authenticated).

Determine entitlement definitions and plan-to-feature mapping with the user before writing catalog resources.

## Security Gate

Run before any catalog writes or app integration:
- Verify app has real authentication. Identify the session principal for billing identity mapping.
- Never use raw emails in identity fields (`owner`, `grantee`, `granteeId`). Prefer: org/tenant id > stable internal user id > non-email `user.id`. Last resort: derive `usr_${hmac_sha256(email, salt)}`.
- Emails are fine for contact fields (e.g. checkout `email`), not identity ids.
- `owner` must be the app's external principal, never Salable internal ids like `owner_...`.
- If auth is missing: stop, recommend auth for the detected stack (e.g. Auth.js for Next.js, Auth0/Clerk for React SPA, Better Auth for Node.js), and do not proceed until auth is in place.
- Public pricing views may remain unauthenticated; checkout, entitlement checks, and subscription management require authenticated identities.

## User Discovery Checklist

Before any provisioning, gather all of the following from the user. Do not assume or skip items.

1. **Features & entitlements** — What gated features? What entitlement keys? (e.g. `export_pdf`, `team_collab`)
2. **Plans** — What plans? (e.g. Free, Starter, Pro) Display names?
3. **Plan-to-entitlement mapping** — Which entitlements does each plan grant? Confirm the full matrix.
4. **Line item types per plan** — Flat-rate? Per-seat? Metered? (Max one `per_seat` per plan.)
5. **Pricing amounts** — Exact amounts per line item. Currency (default USD).
6. **Billing intervals** — Monthly? Yearly? Discounts for longer intervals?
7. **Multi-currency** — Non-USD currencies needed? Amounts?
8. **Seat/usage limits** — Min/max seats? Metered tier breakpoints?

Present a summary table for confirmation before building:

```
Plan    | Entitlements          | Line Items         | Monthly | Yearly
Starter | export, basic_api     | flat_rate $29      | $29/mo  | $290/yr
Pro     | export, full_api, sso | flat_rate + seat   | $49 + $12/seat | $490 + $10/seat
```

## Workflow

1. **Download OpenAPI spec** — `curl -s https://beta.salable.app/openapi.yaml > /tmp/salable-openapi.yaml`
2. **Auth preflight** — Confirm auth exists and identify non-email identity source. Exit if missing.
3. **User discovery** — Walk through checklist. Present summary table for confirmation.
4. **Build in dependency order** via MCP: entitlements -> product -> `plans_save` per plan -> verify with `plans_get`.
5. **Verify** — Re-read persisted prices to confirm amounts match intent. Return resource IDs.
6. **Verify schemas before app code** — Grep `/tmp/salable-openapi.yaml` for exact request/response schemas of every endpoint you will call.
7. **Build app integration** — After catalog provisioning, immediately build the app-side surfaces. Do NOT stop after provisioning and wait for the user to ask. See the App Integration section below.

## App Integration

After catalog provisioning, build all of the following automatically. Read existing files first to avoid duplicating code that already exists; create or update as needed.

### Required deliverables

1. **Salable API utility** — Server-side helper (`salableApi`) with `cache: 'no-store'` and Bearer auth. Identity bridge function (`getSalableIdentity`) mapping the app's session principal to a non-email Salable identity.

2. **Entitlements provider** — Client-side React Context that fetches entitlements on session change and exposes `has(key)`, `loading`, and `refresh()`. Must be mounted in the root layout inside the auth session provider.

3. **API routes** — All route handlers calling Salable MUST include `export const dynamic = 'force-dynamic'`:
   - `GET /api/entitlements` — checks grantee entitlements via Salable REST, returns `{ entitlements }`.
   - `POST /api/checkout` — creates cart -> adds cart-item -> initiates checkout -> returns `{ url }`.
   - `GET /api/subscription` — lists active subscriptions for the authenticated owner.
   - `POST /api/subscription/portal` — creates a Stripe billing portal session, returns `{ url }`.

4. **Pricing page** (public, unauthenticated) — Displays all plans with feature comparison. Free plan links to sign-up or builder. Paid plans trigger checkout (redirect to sign-in first if unauthenticated). Already-subscribed users see "Manage Subscription" linking to account page. Expose the plan ID via `NEXT_PUBLIC_SALABLE_PRO_PLAN_ID` (or similar env var per plan).

5. **Account / subscription management page** (authenticated) — Shows current plan status (entitlement-based), included features list, billing portal button (payment method, invoices, cancellation). Handles `?checkout=success` redirect from Stripe with a success banner and entitlement refresh. Wrap `useSearchParams()` in a `<Suspense>` boundary for Next.js static generation compatibility.

6. **Navigation wiring** — Add Pricing link to the main nav. Add Account link to the authenticated user header area (next to sign out).

7. **Auth protection** — Ensure the account page route is protected by auth middleware. Public pricing page remains unauthenticated.

8. **Environment variables** — Update `.env` and `.env.example` with all required Salable env vars including provisioned plan IDs.

### Build verification

After building, run `next build` to verify compilation. Fix any errors (Suspense boundaries, missing imports, type errors) before presenting the result to the user.

## MCP Tools Reference

| Action | Tool |
|---|---|
| Create/list/update product | `products_create`, `products_list`, `products_update` |
| Create/list entitlements | `entitlements_create`, `entitlements_list` |
| Save complete plan (preferred) | `plans_save` |
| Read/list plans | `plans_get` (with `expand`), `plans_list` |
| Inspect line items/prices | `line_items_list`, `prices_list`, `currency_options_list` |
| Archive resources | `products_archive`, `plans_archive`, `line_items_archive`, `prices_archive` |

All prefixed with `mcp__salable__`. Use `plans_save` as the default write path.

## REST Runtime Endpoints

- **Pricing page** (public): `GET /api/plans`, `GET /api/products`
- **Entitlement check**: `GET /api/entitlements/check`
- **Subscriptions**: `GET /api/subscriptions`, `GET /api/subscriptions/{id}`, `GET /api/subscriptions/{id}/invoices`
- **Billing portal**: `POST /api/subscriptions/{id}/portal`
- **Plan changes**: `PUT /api/subscriptions/{id}/items`
- **Lifecycle**: `POST /api/subscriptions/{id}/cancel`, `PUT /api/subscriptions/{id}/auto-renew`

### Response Wrapping

All REST responses are wrapped in a `data` envelope. Always unwrap through `.data`:

- **Object endpoints** (entitlement check, portal, single GET): `{ "type": "object", "data": { ...payload } }` — access `body.data`.
- **List endpoints** (subscriptions, plans): `{ "type": "list", "data": [ ...items ], "hasMore": bool }` — access `body.data` (the array).

Common mistake: `body.entitlements` instead of `body.data.entitlements`, or `body.url` instead of `body.data.url`.

### Checkout Flow

Three sequential API calls. All required fields must be present or the call returns 400.

**Step 1: Create Cart — `POST /api/carts`**

| Field | Required | Notes |
|---|---|---|
| `owner` | Yes | Your app's billing identity. NOT an email. |
| `interval` | Yes | `day`, `week`, `month`, `year`. Must match plan price. |
| `intervalCount` | Yes | e.g. `1` for monthly. Must match plan price. |
| `currency` | No | ISO currency code. Defaults to plan's default. |

**Step 2: Add Cart Item — `POST /api/cart-items`**

| Field | Required | Notes |
|---|---|---|
| `cartId` | Yes | ID from step 1. |
| `planId` | Yes | The Salable plan ID. |
| `interval` | Yes | Must match cart. |
| `intervalCount` | Yes | Must match cart. |
| `grantee` | No | User receiving entitlements. For solo: same as `owner`. **Goes here, NOT on the cart.** |
| `metadata` | No | JSON object for seat quantities etc. |

**Step 3: Checkout — `POST /api/carts/{id}/checkout`**

| Field | Required | Notes |
|---|---|---|
| `successUrl` | Yes* | *Required unless set in product settings. |
| `cancelUrl` | Yes* | *Required unless set in product settings. |
| `email` | No | Pre-fill Stripe checkout email. |

Returns: `{ "data": { "url": "https://checkout.stripe.com/..." } }` — redirect user to `body.data.url`.

### Entitlement Check — `GET /api/entitlements/check`

Query params: `granteeId` (required), `owner` (optional).

Access entitlements: `body.data.entitlements` — each item has `value` (the key), `type`, and `expiryDate`.

> **The entitlement key is `value`, NOT `name`.** Always map features with `e.value`.

### Billing Portal — `POST /api/subscriptions/{id}/portal`

Creates a Stripe billing portal session. Returns `body.data.url` — redirect the user there.

| Field | Required | Notes |
|---|---|---|
| `features` | **Yes** | Object with **camelCase** keys. Must include at least one sub-feature. |
| `returnUrl` | No | Redirect URL after portal session. |

**`features` sub-features (camelCase only — snake_case causes 500):**

| Key | Required fields |
|---|---|
| `paymentMethodUpdate` | `enabled: boolean` |
| `subscriptionCancel` | `enabled: boolean`, `when: "now" \| "end"` |
| `invoiceHistory` | `enabled: boolean` |
| `customerUpdate` | `enabled: boolean`, `allowedUpdates: ("name"\|"email"\|"address"\|"phone"\|"shipping")[]` |

Each sub-feature has its own required fields — omitting `when` from `subscriptionCancel` or `allowedUpdates` from `customerUpdate` causes 500 errors.

```json
{
  "features": {
    "paymentMethodUpdate": { "enabled": true },
    "subscriptionCancel": { "enabled": true, "when": "end" },
    "invoiceHistory": { "enabled": true }
  },
  "returnUrl": "https://app.example.com/account"
}
```

### Grantee & Group Model

Entitlement access chain: **Grantee -> Membership -> Group -> Subscription Plans -> Entitlements**.

- If `grantee` is passed on the cart-item, Salable auto-creates the grantee and adds them to the subscription's group. Entitlements available immediately.
- If `grantee` is omitted, the group is created empty — manage membership after checkout via `POST /api/groups/{groupId}/grantees` (`add`, `remove`, `replace` operations).
- **Solo user** — pass `grantee` same as `owner`. **Team** — omit `grantee`, manage group post-checkout.

## Safe Update Pattern

When modifying existing plans: fetch with `plans_get`, copy all existing `id` values into the payload, change only intended fields, submit via `plans_save`, re-fetch and compare counts.

## Plan Template (Hybrid)

```json
{
  "name": "Business",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_id_1", "ent_id_2"],
  "lineItems": [
    {
      "name": "base_fee", "slug": "base_fee",
      "priceType": "flat_rate", "intervalType": "recurring", "billingScheme": "flat_rate",
      "minQuantity": 1, "maxQuantity": 1, "tiersMode": null,
      "prices": [{ "defaultCurrency": "USD", "interval": "month", "intervalCount": 1, "currencyOptions": [{ "currency": "USD", "unitAmount": 99 }] }]
    },
    {
      "name": "seat", "slug": "seat",
      "priceType": "per_seat", "intervalType": "recurring", "billingScheme": "per_unit",
      "minQuantity": 1, "maxQuantity": 500, "allowChangingQuantity": true, "tiersMode": null,
      "prices": [{ "defaultCurrency": "USD", "interval": "month", "intervalCount": 1, "currencyOptions": [{ "currency": "USD", "unitAmount": 12 }] }]
    },
    {
      "name": "api_usage", "slug": "api_usage",
      "priceType": "metered", "intervalType": "recurring", "billingScheme": "tiered",
      "tiersMode": "graduated", "meterSlug": "api_usage", "minQuantity": 1, "maxQuantity": 1,
      "prices": [{ "defaultCurrency": "USD", "interval": "month", "intervalCount": 1, "currencyOptions": [{ "currency": "USD", "unitAmount": null, "tiers": [{ "upTo": "10000", "flatAmount": null, "unitAmount": 0.02 }, { "upTo": "inf", "flatAmount": null, "unitAmount": 0.01 }] }] }]
    }
  ]
}
```

For simpler plans, use only the relevant line item types. Add multiple `prices` entries per line item for multi-cadence.

## Guardrails

### Next.js Fetch Caching

Next.js App Router caches `fetch()` by default. Salable API calls MUST disable caching:
- `cache: 'no-store'` on every `fetch()` to Salable.
- `export const dynamic = 'force-dynamic'` on every API route handler calling Salable.

Without both, entitlement checks cached before a subscription keep returning empty forever.

### Pricing and Values

- `unitAmount` is decimal major units (dollars, not cents). $29.00 = `29`, never `2900`. Min non-null: `0.01`.
- Line item slug: `^[a-z0-9_]+$`.
- One `per_seat` line item max per plan.
- Send `lineItems` as objects, not JSON strings.
- Omit `includeArchived` by default. Only send `includeArchived=true` when needed.
- Keep default currency consistent across plans in the same product.
- Do not mix line items with different default currencies in the same cart.
- Provide tiers for every currency option when `billingScheme` is `tiered`.
- Ensure product exists and is not archived before `plans_save`.
- Subscription created only after successful checkout. Updating catalog prices does not migrate existing subscriptions.
- Identity fields reject emails: `owner` (cart body) and `grantee` (cart-item body) — different API calls.
- Entitlement key is `value`, NOT `name`.

## Locked Feature UX Policy

- When a feature is locked by permissions/capabilities, keep the control visible and disabled; do not hide it by default.
- Always explain the lock reason with both a persistent inline note and a hover/focus tooltip.
- Tooltip copy should be short and specific: `Upgrade to unlock this feature.` or `You do not have access to this feature.`
- For disabled buttons, place the tooltip on a non-disabled wrapper so hover/focus still works.
- For locked sections, reduce visual emphasis (muted styling) but keep labels and current values readable.
- For locked views/routes, show a clear unlock message and primary CTA to the upgrade/access path; avoid dead-end screens.
- Keep internal ids/keys out of primary user-facing copy by default; use friendly names in UX text.
- Enforce the same rules server-side. UI locks are guidance; backend authorization is the source of truth.

## Known API/MCP Quirks (Observed)

- `GET /api/plans` and `mcp__salable__plans_list` treat `includeArchived=false` as invalid in some environments. Omit the parameter for active-plan reads; use `includeArchived=true` only when archived plans are explicitly requested.
- `mcp__salable__plans_save` can fail with `Expected object, received string` when `lineItems` are serialized as strings; send object arrays.
- `POST /api/carts/{id}/checkout` can return `400` if `cancelUrl` is not provided; send both `cancelUrl` and `successUrl` by default.
- `POST /api/carts`, `POST /api/cart-items`, and related identity fields can fail validation when `owner`/`grantee` values are email-formatted strings.
- If application login works but runtime billing endpoints return `Unauthenticated`, suspect Salable API credential/config issues first (`SALABLE_API_KEY` placeholder/invalid, wrong `SALABLE_API_BASE_URL`) before debugging session auth.
- Persisted `currencyOptions.unitAmount` values may appear in minor units in responses even when plan-write payloads are provided as major decimal units; always re-read and validate displayed prices explicitly.
- `subscription.ownerId` can be an internal owner reference; owner checks for user-initiated actions should use owner-scoped list queries (`owner` + `id`) rather than direct equality with application principal id.

## References

For additional detail, read these reference files in the plugin's `references/` directory:
- `references/mcp-tool-playbook.md` — operation-by-operation MCP guidance.
- `references/pricing-model-templates.md` — ready-to-adapt payload patterns.
- `references/openapi-focus.md` — API endpoints that map to MCP operations.
- `references/auth-options.md` — stack/language authentication recommendations and exit-response wording.
