---
name: happy-cabbage-inventory-health-review
description: >-
  Review a cannabis retailer's inventory health in Happy Buyers across stores, brands and categories,
  identify where inventory is too deep or too thin, and hand back a prioritised list. Read-only.
api: Happy Buyers External API
base_url: https://api.happycabbage.ai
auth: API key in the hca-api-key header
required_scopes:
  - organization_metadata:read
  - inventory:read
operations:
  - whoami
  - getStores
  - getStoreInventoryHealths
  - getStoreInventoryHealthHistories
  - getCategoryInventoryHealths
  - getPosBrandInventoryHealths
  - getProductInventory
generated: '2026-08-22'
method: generated
source: openapi/happy-cabbage-analytics-happy-buyers-external-openapi.yml
---

# Inventory health review

Every operationId below was read from the Happy Buyers External API contract. Nothing here writes.

## 1. Confirm the tenant

Call `whoami` (`GET /external/v1/whoami`). It returns `keyName`, `organizationName`, `keyCreatedAt` and
`organizationCreatedAt`. State the organization name back to the user before doing anything else — every
result you are about to read is scoped to whatever organization that key belongs to, and the key does not
tell you which one that is by looking at it.

## 2. Enumerate the stores

Call `getStores` (`GET /external/v1/stores`). Paginate with `limit` and `offset`; keep going while
`hasMore` is true. `totalCount` tells you how many stores exist before you start.

## 3. Read inventory health at each level

- `getStoreInventoryHealths` — `GET /external/v1/inventory-health/stores`
- `getCategoryInventoryHealths` — `GET /external/v1/inventory-health/categories`
- `getPosBrandInventoryHealths` — `GET /external/v1/inventory-health/pos-brands`

Filter with `storeIds` (repeat the query parameter per store). Use `includePercentShares` when you want
share-of-inventory alongside the absolute numbers, and `lowDepthOnly` to narrow straight to the lines that
are under-assorted.

## 4. Add the trend

For anything that looks wrong at a point in time, pull the matching history operation —
`getStoreInventoryHealthHistories`, `getCategoryInventoryHealthHistories`,
`getPosBrandInventoryHealthHistory` — with `periodType` to choose the grain. A single week of bad
days-on-hand is noise; four in a row is a buying problem.

## 5. Drill to products

`getProductInventory` (`GET /external/v1/product-inventory`) carries the per-product picture. The range
filters are how you find the tails: `maxDaysAged` / `minDaysAged` for aging, `minimumAgedInventoryCost` for
dead capital, `maximumPredictedDaysOnHand` for products about to stock out, `minimumPredictedDaysOnHand`
for products drowning. `search`, `matchingSearchTerms` and `notMatchingSearchTerms` narrow by name.

## 6. Report

Give the user a ranked list: store, brand or category, the metric that is out of band, the direction, and
the trend. Do not propose an order in this skill — that is
`happy-cabbage-analytics-draft-replenishment-order`.

## Conventions that apply throughout

- Pagination is `limit` / `offset`; the response envelope is `{limit, offset, totalCount, hasMore, results}`.
- A 401 means the `hca-api-key` header is missing, malformed or invalid. A 403 names the scope the key is
  missing — read the response description, it tells you exactly which one.
- There are no documented rate limits and no `Retry-After` header. Pace yourself and back off on any
  non-2xx you did not expect rather than retrying tightly.
