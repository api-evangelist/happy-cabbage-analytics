---
name: happy-cabbage-manage-product-lines
description: >-
  Create and tune Happy Buyers product lines (demand groups) — the grouping that drives replenishment
  targets — and set which products a retailer continues to carry. Writes; product lines cannot be deleted.
api: Happy Buyers External API
base_url: https://api.happycabbage.ai
auth: API key in the hca-api-key header
required_scopes:
  - organization_metadata:read
  - product_lines:read
  - product_lines:write
  - inventory:read
  - inventory:write
operations:
  - whoami
  - getUniversalBrands
  - getUniversalCategories
  - getPosBrands
  - getPosCategories
  - getProductLines
  - getProductLine
  - createProductLine
  - updateProductLine
  - getProductLineProductInventory
  - updateCarryStatus
generated: '2026-08-22'
method: generated
source: openapi/happy-cabbage-analytics-happy-buyers-external-openapi.yml
---

# Manage product lines and carry status

A product line in Happy Buyers is a demand group: a named query over products that carries its own
`desiredDaysOnHand`, `leadTime`, `lookbackPeriod` and depth target, and drives the replenishment maths for
everything inside it. Getting these wrong quietly distorts every order that follows, so this skill is
deliberate about verification.

## Safety first

`createProductLine` (`POST /external/v1/product-lines`) has **no idempotency key and no delete
operation**. A duplicate line created by a retry stays in the account until a human removes it in the app,
and while it exists it competes for the same products. Confirm before creating, and if a create call
fails ambiguously, call `getProductLines` and search by name before retrying.

`updateCarryStatus` is the one genuinely safe write here: setting `continueToCarry: false` discontinues a
product from reorder recommendations, the product remains viewable, and setting it back to `true` fully
restores it.

## 1. Establish the tenant and the vocabulary

`whoami`, then read the canonical vocabularies before you build a query:

- `getUniversalBrands` (`GET /external/v1/universal-brands`) and `getUniversalCategories`
  (`GET /external/v1/universal-categories`) — Happy Cabbage's cross-POS canonical ids.
- `getPosBrands` and `getPosCategories` — the raw POS-specific labels and their mappings.

Prefer universal ids when the retailer runs more than one POS; a POS brand id means nothing outside the
system it came from.

## 2. Inspect what exists

`getProductLines` (`GET /external/v1/product-lines`) with `includeProductLineDetails` to see the query
parameters and targets on each line. `getProductLine` (`GET /external/v1/product-lines/{id}`) for one.
`getProductLineProductInventory` (`GET /external/v1/product-lines/{id}/product-inventory`) shows exactly
which products a line currently captures — always run this before and after a change so you can show the
user what moved.

## 3. Create or update

`createProductLine` / `updateProductLine` take a `name` plus `productQueryParams` (the membership query),
and optionally `desiredDaysOnHand` (minimum 1), `leadTime` (minimum 0), `lookbackPeriod` (minimum 1),
`isFavorite`, `autoRunDays` (ISO day-of-week integers, default Monday–Friday) and
`reportEmailTargetDefaults`.

`updateProductLine` replaces the definition — `name` and `productQueryParams` are both required, so read
the current line first and carry forward anything you are not deliberately changing.

## 4. Set carry status

`updateCarryStatus`
(`PUT /external/v1/product-inventory/{productId}/stores/{storeId}/carry-status`) with
`{"continueToCarry": false}` discontinues a product at one store. It is per-store — loop the stores you
mean, and do not assume a chain-wide effect.

## 5. Verify

Re-run `getProductLineProductInventory` and report the before/after membership and target values.

## Errors

`400` invalid body, `401` bad or missing `hca-api-key`, `403` missing scope (`product_lines:write` or
`inventory:write` — the description names it), `404` product line, store or product inventory not found.
