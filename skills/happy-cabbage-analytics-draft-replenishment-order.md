---
name: happy-cabbage-draft-replenishment-order
description: >-
  Build a draft purchase order in Happy Buyers from replenishment signals — find what needs reordering,
  create the order, add line items, and attach the vendor invoice. Writes. Requires human confirmation.
api: Happy Buyers External API
base_url: https://api.happycabbage.ai
auth: API key in the hca-api-key header
required_scopes:
  - organization_metadata:read
  - inventory:read
  - product_lines:read
  - orders:write
operations:
  - whoami
  - getStores
  - getProductLineInventory
  - getProductInventory
  - getBlockoutDates
  - createOrder
  - addOrderItem
  - getOrderItems
  - removeOrderItem
  - updateOrder
  - addOrderInvoice
  - generateTemporaryUploadUrl
  - removeOrderInvoice
generated: '2026-08-22'
method: generated
source: openapi/happy-cabbage-analytics-happy-buyers-external-openapi.yml
---

# Draft a replenishment order

This skill writes. Read the safety section before the first `POST`.

## Safety first — what you cannot take back

- `createOrder` (`POST /external/v1/orders`) has **no idempotency key**. If the call times out and you
  retry, you create a second purchase order. There is also **no cancel, void or delete operation for an
  order** in the external contract, and `updateOrder` only changes `note`, `name` and `description` — the
  `status` field (`DRAFT`, `READY_TO_SEND`, `SUBMITTED`, `RECEIVED`) is read-only here. An order you create
  in error has to be cleaned up by a human inside Happy Buyers.
- Therefore: **confirm with the user before calling `createOrder`**, and if a create call fails without a
  clear response, call `getOrders` and look for the order before retrying.
- Line items and invoices *are* reversible — `removeOrderItem` and `removeOrderInvoice` undo `addOrderItem`
  and `addOrderInvoice`. Happy Cabbage states no time window on either, so treat "soon" as the only safe
  assumption.

## 1. Establish the tenant

`whoami`, then `getStores`. Name the organization and the store(s) you are ordering for back to the user.

## 2. Find what needs replenishing

- `getProductLineInventory` (`GET /external/v1/product-line-inventory`) is the primary signal: product
  lines carry `desiredDaysOnHand`, `leadTime` and depth, and the response carries the demand and
  replenishment metrics computed from them. Use `lowDepthOnly` and `restockOnly` to narrow.
- `getProductInventory` (`GET /external/v1/product-inventory`) for the SKU-level view. Use
  `maximumPredictedDaysOnHand` to surface imminent stockouts and `isInStock` / `continueToCarry` to skip
  products the retailer has already discontinued.
- Filter to the stores you established in step 1 with repeated `storeIds` parameters.

## 3. Check blockout dates

`getBlockoutDates` (`GET /external/v1/blockout-dates`). If the delivery window you are planning falls on a
blockout date, say so and stop rather than ordering into it.

## 4. Create the order

`createOrder` (`POST /external/v1/orders`) with `name`, `description` and `note`. It returns an
`OrderResponse` with `id`, `orderNumber` (for example `HB-2026-000123`) and `status`. Record the `id` —
you need it for every following call, and it is your only handle on an order you cannot delete.

## 5. Add the line items

`addOrderItem` (`PUT /external/v1/orders/{orderId}/items`) once per line. Verify with `getOrderItems`
(`GET /external/v1/orders/{orderId}/items`) and remove any mistake with `removeOrderItem`
(`DELETE /external/v1/orders/{orderId}/items/{itemId}/store/{storeId}` — note it needs the store id too).

## 6. Attach the invoice, if you have one

`addOrderInvoice` (`POST /external/v1/orders/{orderId}/invoices`) creates the invoice record, then
`generateTemporaryUploadUrl` (`POST /external/v1/orders/{orderId}/invoices/{invoiceId}/upload-url`) returns
a URL you PUT the file to. The contract calls the URL temporary but does not state its lifetime, so use it
immediately. `generateTemporaryDownloadUrl` is the read side. `updateOrderInvoice` edits the record and
`removeOrderInvoice` deletes it.

## 7. Hand back

Report the `orderNumber`, the store(s), the line count and the total. The order stays in `DRAFT` — sending
it to the vendor happens in Happy Buyers, not through this API.

## Errors

- `400` — a query parameter or body field is invalid. Fix it; do not retry unchanged.
- `401` — the `hca-api-key` header is missing, malformed or invalid.
- `403` — the key lacks a scope; the response description names it (`orders:write` for everything in
  steps 4 to 6).
- `404` — the order, product line, store or product does not exist in this organization.
