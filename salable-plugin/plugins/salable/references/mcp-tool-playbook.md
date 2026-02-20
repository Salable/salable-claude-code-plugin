# Salable MCP Tool Playbook

## Primary References

- Docs: `https://beta.salable.app/docs`
- Local OpenAPI (preferred): `references/openapi.yaml`
- OpenAPI: `https://beta.salable.app/openapi.yaml`

## Version Boundary

- Use only `beta.salable.app` references for Salable 2.0 work.
- Do not use `docs.salable.app` or non-beta `salable.app` references in this workflow.

## Default Build Sequence

1. Create or find product.
- Create: `mcp__salable__products_create`
- Reuse: `mcp__salable__products_list`
- Establish product-level default currency convention for plans on that product.

2. Create or find entitlements.
- Create: `mcp__salable__entitlements_create`
- Reuse: `mcp__salable__entitlements_list`
- Determine entitlement names and plan-to-feature mapping with the user before creating/updating entitlements.

3. Create or update the plan and pricing model.
- Preferred: `mcp__salable__plans_save`
- Use `entitlements` to attach entitlement IDs to the plan.
- For updates, load current plan data first and carry forward nested IDs for objects being modified.
- Add multiple price entries per line item when the plan supports multiple interval/intervalCount cadences.

4. Inspect generated resources.
- Plan detail: `mcp__salable__plans_get` with `expand`
- Line items: `mcp__salable__line_items_list`
- Prices: `mcp__salable__prices_list`
- Currency options: `mcp__salable__currency_options_list`

5. Adjust plan metadata when needed.
- Update plan identity or status: `mcp__salable__plans_update`

6. Archive or retire old resources deliberately.
- Archive product: `mcp__salable__products_archive`
- Archive plan: `mcp__salable__plans_archive`
- Archive line item: `mcp__salable__line_items_archive`
- Archive price: `mcp__salable__prices_archive`

## MCP vs REST Decision

- Use MCP for pricing catalog provisioning and configuration writes.
- Use REST for runtime app views and user-facing flows; app runtime integration should not fallback to MCP calls.
- Validate REST payload constraints against `references/openapi.yaml` first.
- If the local file is missing or stale, validate against `https://beta.salable.app/openapi.yaml` and refresh the local copy.
- If hosted OpenAPI fetch is blocked by sandbox/network policy, request user permission before fetching.
- If permission is denied, continue with the documented REST endpoint set and explicitly note that hosted OpenAPI refresh was denied.
- If OpenAPI is unavailable, continue with the documented REST endpoint set in this skill and note the assumption explicitly.
- Run auth preflight first. If auth is missing, exit and recommend stack-specific auth options from `references/auth-options.md`.
- During auth preflight, require non-email identity mapping for Salable fields (`owner`, `grantee`, `granteeId`).
- If only email is available, derive a deterministic salted hash id and use that id consistently as the billing principal.

## Runtime App Views (REST-First)

1. Pricing table/store page.
- Keep the pricing/paywall route public (no auth required).
- Use `GET /api/plans` with query filters for interval/currency/product.
- Use `GET /api/products` for product metadata.
- When a product has region-specific exception plans with different default currency, filter plans by target market/currency in the UI.

2. Entitlement checks.
- Use `GET /api/entitlements/check` to resolve feature access for a grantee.

3. Subscription management views.
- Use `GET /api/subscriptions` for listing.
- Use `GET /api/subscriptions/{id}` for details.
- Use `GET /api/subscriptions/{id}/invoices` for billing history.
- Use `POST /api/subscriptions/{id}/portal` for hosted management URL.
- Add a visible Stripe billing portal link/button in the UI that opens the URL returned by `POST /api/subscriptions/{id}/portal`.
- Use `PUT /api/subscriptions/{id}/items` for add/remove/replace plan operations.
- Use `POST /api/subscriptions/{id}/cancel` and `PUT /api/subscriptions/{id}/auto-renew` for lifecycle controls.

4. Store checkout flow.
- Use `POST /api/carts` to create cart context.
- Use `POST /api/cart-items` to attach plans.
- Use `POST /api/carts/{id}/checkout` to get checkout URL.
- Use non-email ids for `owner` and `grantee`; do not send raw email addresses in those fields.
- Always send both `cancelUrl` and `successUrl` for checkout link generation; some environments return `400` when `cancelUrl` is omitted.
- Set cart `interval` and `intervalCount` deliberately because checkout matches these fields to plan prices and pulls only matching line item prices.
- Ensure all cart line items share the same default currency; mixed default currencies can break Stripe geolocation and may block checkout.

## Observed Quirks and Workarounds

- `mcp__salable__plans_list` and `GET /api/plans` should omit `includeArchived` for normal active-plan reads. Never send `includeArchived=false`; use `includeArchived=true` only when archived plans are explicitly required.
- `mcp__salable__plans_save` expects `lineItems` as objects; do not serialize line items as JSON strings.
- A `pending` payment integration can allow catalog writes but checkout/billing actions may still fail until onboarding completes.
- Email-formatted values in `owner`/`grantee` identity fields can fail validation; use stable non-email ids.

## Subscription and Billing Notes

- Subscription is created only after successful checkout.
- Billing anchor is fixed at subscription creation.
- All plans inside a subscription share the same billing cycle.
- Updating plan catalog prices does not auto-migrate existing subscriptions.
- When changing plans with metered items, outstanding usage may be finalized/invoiced and counters reset for the replacement configuration.

## Verification Checklist

- Confirm target `productId` exists and is not archived.
- Confirm entitlement mapping was explicitly confirmed with the user.
- Confirm expected entitlements are present on the plan.
- Confirm interval and intervalCount match intended billing cadence.
- Confirm each line item has expected priceType and billingScheme.
- Confirm the plan contains no more than one `per_seat` line item.
- Confirm update payload keeps existing IDs: plan, line items, prices, currency options, and tiers.
- Confirm each supported cadence (for example month/1, month/3, year/1, week/2) exists on required line items.
- Confirm checkout cadence selection returns only the expected matching line item prices.
- Confirm currency options and tier values are correctly persisted.
- Confirm every non-null `unitAmount` is at least `0.01`.
- Confirm default currency is consistent across plans on the same product unless an intentional regional exception is defined.
- Confirm any exception plan (for example UK-only GBP) is intentionally scoped and surfaced only to the correct audience.
- Confirm all line items in a cart resolve to the same default currency before checkout.
- Confirm subscription-management flows account for fixed billing anchor and shared cycle behavior.
- Confirm auth exists before any provisioning or integration work; otherwise exit with stack-specific recommendations.
