# Pricing Model Templates

## Notes

- Adapt these examples to the exact runtime schema exposed by `mcp__salable__plans_save`.
- Keep line item slugs lowercase with underscores.
- Use decimal major units for `unitAmount` and tier amounts.
- For non-null values, `unitAmount` minimum is `0.01`.
- Plans can include multiple line items with mixed `priceType` values (`flat_rate`, `per_seat`, `metered`).
- Use at most one `per_seat` line item per plan.
- Plans can also include multiple interval/intervalCount price cadences in the same plan.
- For `plans_save` updates, include IDs for existing objects (`id` on plan, line items, prices, currency options, tiers) or new duplicates may be created.
- Checkout/cart flow matches by `interval` + `intervalCount` and uses only line item prices matching that cadence.
- Default currency should usually be consistent across plans on a product.
- Default-currency exceptions are allowed for intentional regional plans (for example UK-only `GBP` plan on an otherwise `USD` product).
- Cart checkout should not mix line items with different default currencies.
- Existing subscriptions remain on their current price version until explicitly migrated.
- For active-plan listing, omit `includeArchived`; never send `includeArchived=false` (use `includeArchived=true` only when archived plans are intentionally requested).
- For checkout link generation, send both `cancelUrl` and `successUrl` by default to avoid environment-specific `400` responses.

## Template 1: Single Flat Monthly Plan

Use for one base subscription that grants a fixed entitlement set.

```json
{
  "name": "Starter",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_projects_basic", "ent_export_png"],
  "lineItems": [
    {
      "name": "starter_base",
      "slug": "starter_base",
      "priceType": "flat_rate",
      "intervalType": "recurring",
      "billingScheme": "flat_rate",
      "minQuantity": 1,
      "maxQuantity": 1,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [
            {
              "currency": "USD",
              "unitAmount": 29.00
            }
          ]
        }
      ]
    }
  ]
}
```

## Template 1b: Single Plan with Monthly, Quarterly, and Yearly Cadences

Use when one plan should support multiple billing cadences and checkout should select by cadence.

```json
{
  "name": "Starter Multi Cadence",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_projects_basic", "ent_export_png"],
  "lineItems": [
    {
      "name": "starter_base",
      "slug": "starter_base",
      "priceType": "flat_rate",
      "intervalType": "recurring",
      "billingScheme": "flat_rate",
      "minQuantity": 1,
      "maxQuantity": 1,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "USD", "unitAmount": 29.00 }]
        },
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 3,
          "currencyOptions": [{ "currency": "USD", "unitAmount": 81.00 }]
        },
        {
          "defaultCurrency": "USD",
          "interval": "year",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "USD", "unitAmount": 290.00 }]
        }
      ]
    }
  ]
}
```

Checkout examples:
1. Cart `interval=month`, `intervalCount=1` -> monthly price is selected.
2. Cart `interval=month`, `intervalCount=3` -> quarterly price is selected.
3. Cart `interval=year`, `intervalCount=1` -> yearly price is selected.

## Template 1c: Regional Default Currency Exception Plan

Use when most plans for a product default to one currency (for example `USD`), but one plan is intentionally regional (for example UK-only with `GBP` default).

```json
{
  "name": "Starter UK",
  "productId": "prod_same_as_usd_product",
  "isActive": true,
  "entitlements": ["ent_projects_basic", "ent_export_png"],
  "lineItems": [
    {
      "name": "starter_base_uk",
      "slug": "starter_base_uk",
      "priceType": "flat_rate",
      "intervalType": "recurring",
      "billingScheme": "flat_rate",
      "minQuantity": 1,
      "maxQuantity": 1,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "GBP",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "GBP", "unitAmount": 24.00 }]
        },
        {
          "defaultCurrency": "GBP",
          "interval": "year",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "GBP", "unitAmount": 240.00 }]
        }
      ]
    }
  ]
}
```

Guidance:
1. Keep this exception explicit in naming and rollout notes.
2. Ensure app/store filtering shows this plan only to intended UK audience.

## Template 2: Per-Seat Team Plan

Use for seat-based licensing with configurable quantity.

```json
{
  "name": "Team",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_team_collab", "ent_sso"],
  "lineItems": [
    {
      "name": "team_seat",
      "slug": "team_seat",
      "priceType": "per_seat",
      "intervalType": "recurring",
      "billingScheme": "per_unit",
      "minQuantity": 3,
      "maxQuantity": 500,
      "allowChangingQuantity": true,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [
            {
              "currency": "USD",
              "unitAmount": 12.00
            }
          ]
        }
      ]
    }
  ]
}
```

## Template 3: Metered Tiered Usage

Use for usage-based billing where price changes by consumption tiers.

```json
{
  "name": "Usage Pro",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_api_access", "ent_usage_dashboard"],
  "lineItems": [
    {
      "name": "api_calls",
      "slug": "api_calls",
      "priceType": "metered",
      "intervalType": "recurring",
      "billingScheme": "tiered",
      "tiersMode": "graduated",
      "meterSlug": "api_calls",
      "minQuantity": 1,
      "maxQuantity": 1,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [
            {
              "currency": "USD",
              "unitAmount": null,
              "tiers": [
                { "upTo": "10000", "flatAmount": null, "unitAmount": 0.03 },
                { "upTo": "50000", "flatAmount": null, "unitAmount": 0.02 },
                { "upTo": "inf", "flatAmount": null, "unitAmount": 0.01 }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

## Template 4: Add Entitlement to Existing Plan

1. Read existing plan with `mcp__salable__plans_get`.
2. Create missing entitlement with `mcp__salable__entitlements_create`.
3. Re-submit plan through `mcp__salable__plans_save` with existing IDs preserved.
4. Re-read plan and verify the new entitlement is attached.

## Template 5: Hybrid Plan (Base + Seats + Metered Overage)

Use when one plan needs a platform fee, seat billing, and usage overage.

```json
{
  "name": "Business Hybrid",
  "productId": "prod_...",
  "isActive": true,
  "entitlements": ["ent_team_collab", "ent_advanced_reports", "ent_api_access"],
  "lineItems": [
    {
      "name": "base_platform_fee",
      "slug": "base_platform_fee",
      "priceType": "flat_rate",
      "intervalType": "recurring",
      "billingScheme": "flat_rate",
      "minQuantity": 1,
      "maxQuantity": 1,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "USD", "unitAmount": 99.00 }]
        }
      ]
    },
    {
      "name": "member_seat",
      "slug": "member_seat",
      "priceType": "per_seat",
      "intervalType": "recurring",
      "billingScheme": "per_unit",
      "minQuantity": 5,
      "maxQuantity": 500,
      "allowChangingQuantity": true,
      "tiersMode": null,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [{ "currency": "USD", "unitAmount": 15.00 }]
        }
      ]
    },
    {
      "name": "api_calls_overage",
      "slug": "api_calls_overage",
      "priceType": "metered",
      "intervalType": "recurring",
      "billingScheme": "tiered",
      "tiersMode": "graduated",
      "meterSlug": "api_calls_overage",
      "minQuantity": 1,
      "maxQuantity": 1,
      "prices": [
        {
          "defaultCurrency": "USD",
          "interval": "month",
          "intervalCount": 1,
          "currencyOptions": [
            {
              "currency": "USD",
              "unitAmount": null,
              "tiers": [
                { "upTo": "50000", "flatAmount": null, "unitAmount": 0.02 },
                { "upTo": "inf", "flatAmount": null, "unitAmount": 0.01 }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

## Safe Update Pattern (ID-Preserving)

1. Fetch existing plan with `mcp__salable__plans_get` and include nested fields you will edit.
2. Copy existing `id` values into the update payload for plan, line items, prices, currency options, and tiers.
3. Change only intended fields (amounts, names, intervals, entitlement list).
4. Submit with `mcp__salable__plans_save`.
5. Re-fetch and compare counts to ensure no unintended duplicates were created.
