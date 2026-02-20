# OpenAPI Focus Map

## Contract Sources

- Primary local spec (preferred): `references/openapi.yaml`
- Primary hosted spec: `https://beta.salable.app/openapi.yaml`
- Primary docs: `https://beta.salable.app/docs`
- If the local spec is missing or stale, refresh from hosted OpenAPI.
- If hosted fetch is blocked by sandbox/network policy, request user permission before fetching.
- If permission is denied, continue with this REST endpoint map and explicitly note that hosted OpenAPI refresh was denied.
- If hosted OpenAPI is unavailable, keep app runtime work REST-first using this endpoint map and document the missing-spec assumption.
- If app authentication is missing, exit first and recommend stack-specific auth options from `references/auth-options.md`.

## Version Boundary

- Use only `beta.salable.app` contracts for Salable 2.0 work.
- Do not use `docs.salable.app` or non-beta `salable.app` in this map.

## Core Pricing Endpoints

- Products: `/api/products`
- Plans: `/api/plans`
- Full plan save (entitlements + line items + prices): `/api/plans/save`
- Entitlements: `/api/entitlements`
- Line items: `/api/line-items`
- Prices: `/api/prices`
- Currency options: `/api/currency-options`
- Tiers: `/api/tiers`
- Meters: `/api/meters`

## Runtime App Endpoints

- Pricing/store pages: `/api/plans`, `/api/products`
- Pricing/paywall route should remain public; auth is required for checkout, entitlement, and subscription-management actions.
- Entitlement checks: `/api/entitlements/check`
- Subscription management views: `/api/subscriptions`, `/api/subscriptions/{id}`, `/api/subscriptions/{id}/invoices`, `/api/subscriptions/{id}/portal`, `/api/subscriptions/{id}/items`, `/api/subscriptions/{id}/cancel`, `/api/subscriptions/{id}/auto-renew`
- Subscription management UI requirement: include a Stripe billing portal link/button that opens the URL from `/api/subscriptions/{id}/portal`.
- Cart and checkout: `/api/carts`, `/api/cart-items`, `/api/carts/{id}/checkout`
- Use non-email identity values for `owner` and `grantee` when calling cart endpoints.
- Checkout link generation should include both `cancelUrl` and `successUrl` to avoid environment-specific `400` errors.
- Checkout matching key: cart `interval` + `intervalCount` against plan price cadence.
- Checkout currency constraint: all line items in a cart should share one default currency.

## Endpoint to MCP Mapping

- `/api/products` -> `mcp__salable__products_create`, `mcp__salable__products_list`, `mcp__salable__products_update`
- `/api/plans` -> `mcp__salable__plans_create`, `mcp__salable__plans_list`, `mcp__salable__plans_update`
- `/api/plans/save` -> `mcp__salable__plans_save`
- `/api/entitlements` -> `mcp__salable__entitlements_create`, `mcp__salable__entitlements_list`
- `/api/line-items` -> `mcp__salable__line_items_list`, `mcp__salable__line_items_get`
- `/api/prices` -> `mcp__salable__prices_list`, `mcp__salable__prices_get`
- `/api/currency-options` -> `mcp__salable__currency_options_list`, `mcp__salable__currency_options_get`

## REST Runtime Focus

- Prefer these REST endpoints when building app UI views and user flows.
- Keep MCP operations focused on catalog/provisioning automation.
- Do not replace runtime REST integration with MCP calls.
- Do not proceed with runtime implementation when authentication is absent.

## Fields to Double-Check

- Entitlement naming and plan-to-feature mapping are confirmed with the user.
- `lineItems[].slug` follows slug format constraints.
- `priceType`, `intervalType`, and `billingScheme` are internally consistent.
- `tiersMode` is set only when `billingScheme` is `tiered`.
- `interval` and `intervalCount` reflect intended cadence.
- Non-null `unitAmount` values should be `>= 0.01`.
- Plans can include multiple interval and intervalCount combinations across prices in the same plan.
- Default currency is typically aligned across plans within a product.
- Intentional regional exceptions (for example UK-only plan with `GBP` default on a `USD` product) are allowed.
- Mixed default currencies in one cart can break Stripe geolocation and may block checkout.
- Entitlement names and IDs are stable and reusable across plans.
- Billing identity mapping uses non-email ids for `owner`/`grantee`; if only email exists, use a deterministic salted-hash id.
- For `plans_save` updates, preserve existing IDs across nested objects to avoid duplicated records.
- If no auth exists, stop the workflow and return stack/language-specific auth recommendations.
- For plan listing, omit `includeArchived` for active-plan reads. Never send `includeArchived=false`; use `includeArchived=true` only when archived plans are intentionally requested.
- For `plans_save`, ensure `lineItems` are object arrays and not serialized JSON strings.
