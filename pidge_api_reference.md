# Order Tracking — Migration Brief
> **Migrating from:** Scroll Engine
> **Migrating to:** Pidge
> **Related files:** `scroll_engine_tracking.md` | `pidge_api_reference.md`

---

## 1. Provider Comparison

| | Scroll Engine (current) | Pidge (new) |
|---|---|---|
| Auth | None — `shop_url` based | Bearer token (JWT via login) |
| Initial load | `trackOrder` — poll until `status=true` | `GET /order/:id` |
| Live tracking | `trackShipment` — poll every 8s | `GET /order/:id/fulfillment/tracking` + Webhook push |
| Status updates | Pull only | **Push via Webhook** (+ pull available) |
| Driver GPS | Inside `trackShipment` response | Separate `/fulfillment/tracking` endpoint |
| Delivery timestamp | `history_timeline[].updatedAt` | `fulfillment.drop.timestamp` (webhook) |
| Status label text | Provided by API (`status_text.*`) | **Not provided — must generate on our side** |
| ETA string | Provided as string (`status_text.eta`) | ISO timestamp (`fulfillment.drop.eta`) — must format |
| Driver feedback | `getFeedback` + submit API | **Not documented — needs clarification** |
| Multi-shipment | `all_shipments_info[]` | **Not documented — needs clarification** |
| Public tracking URL | `shipment.trackingInfo.url` | `fulfillment.track_code` — needs to confirm if full URL |
| Payment data | `payment.payment_link`, `outstanding_amount` | Not present — stays in our system |
| Third-party detection | `assigned_to.email == "hello@home-run.co"` | Not applicable — logic to be removed |

---

## 2. API Mapping — Scroll Engine → Pidge

### 2a. trackOrder → Get Order Status

| | Scroll Engine | Pidge |
|---|---|---|
| Method | POST | GET |
| Endpoint | `/trackOrder` | `/v1.0/store/channel/vendor/order/:id` |
| Request | `{ shop_url, order_id }` | `Authorization: Bearer <token>` |
| Order ID | Shopify numeric ID | Pidge's own order ID (assigned at creation) |

---

### 2b. trackShipment → Get Rider Location + Webhook

| | Scroll Engine | Pidge |
|---|---|---|
| Method | POST | GET |
| Endpoint | `/trackShipment` | `/v1.0/store/channel/vendor/order/:id/fulfillment/tracking` |
| Request | `{ shop_url, shipment_id }` | `Authorization: Bearer <token>` |
| Polling | Every 8s (app-side loop) | Optional — Webhook replaces polling |
| Driver GPS | `driver_geo_location.lat/lng` | `data.location.latitude/longitude` |
| Driver name | `assigned_to.first_name + last_name` | `data.rider.name` (single full name string) |
| Driver phone | `assigned_to.phone` | `data.rider.mobile` |
| Status | `shipment.shipment_status` | `data.status` |

---

### 2c. Status Updates → Webhook (new)

Pidge pushes to our callback URL in real time. This can replace the 8s polling loop.

**Webhook payload key fields:**

| Field | Type | Replaces |
|---|---|---|
| `fulfillment.status` | String | `shipment.shipment_status` |
| `fulfillment.logs[].status` | String | `history_timeline[].status` |
| `fulfillment.logs[].timestamp` | String (ISO UTC) | `history_timeline[].updatedAt` |
| `fulfillment.drop.timestamp` | String (ISO UTC) | Delivery confirmed time |
| `fulfillment.drop.eta` | String (ISO UTC) | `status_text.eta` (was a string, now a timestamp — must format) |
| `fulfillment.pickup.location.lat/lng` | Double | `shipment.start_location.order_location.lat/lng` |
| `fulfillment.drop.location.lat/lng` | Double | `shipment.end_location.order_location.lat/lng` |
| `fulfillment.rider.name` | String | `assigned_to.first_name + last_name` |
| `fulfillment.rider.mobile` | String | `assigned_to.phone` |

---

## 3. Field-by-Field Mapping

| What our UI needs | Scroll Engine key | Pidge equivalent | Gap |
|---|---|---|---|
| Shipment ID | `shipment._id` | `id` (Pidge order ID) | None |
| Shipment status | `shipment.shipment_status` | `fulfillment.status` / `data.status` | Status values differ — see Section 4 |
| Store lat | `shipment.start_location.order_location.lat` | `fulfillment.pickup.location.latitude` (webhook) | None |
| Store lng | `shipment.start_location.order_location.lng` | `fulfillment.pickup.location.longitude` (webhook) | None |
| Customer lat | `shipment.end_location.order_location.lat` | `fulfillment.drop.location.latitude` (webhook) | None |
| Customer lng | `shipment.end_location.order_location.lng` | `fulfillment.drop.location.longitude` (webhook) | None |
| Driver live GPS lat | `driver_geo_location.latitude` | `data.location.latitude` from `/fulfillment/tracking` | None |
| Driver live GPS lng | `driver_geo_location.longitude` | `data.location.longitude` from `/fulfillment/tracking` | None |
| Driver name | `assigned_to.first_name + last_name` | `data.rider.name` (full string) | Minor — single vs split name |
| Driver phone | `assigned_to.phone` | `data.rider.mobile` | None |
| Main status label | `status_text.order` | ❌ Not provided | **We must generate from status enum** |
| ETA text | `status_text.eta` (string) | `fulfillment.drop.eta` (ISO timestamp) | **Must format timestamp ourselves** |
| Confirmation message | `status_text.order_confirmation_msg` | ❌ Not provided | **We must generate** |
| Driver message | `status_text.driver` | ❌ Not provided | **We must generate** |
| Delivery timestamp | `history_timeline[i].updatedAt` | `fulfillment.drop.timestamp` (webhook) | None |
| Public tracking URL | `shipment.trackingInfo.url` | `fulfillment.track_code` | **Unclear — full URL or code only?** |
| Payment link | `payment.payment_link` | ❌ Not in Pidge | Stays in our system |
| Outstanding amount | `payment.outstanding_amount` | ❌ Not in Pidge | Stays in our system |
| AWB number | `shipment.awb` | ❌ Not documented | **Needs clarification** |
| Multi-shipment list | `all_shipments_info[]` | ❌ Not documented | **Needs clarification** |
| Third-party flag | `assigned_to.email` | ❌ Not applicable | **Remove this logic** |
| Driver feedback | `shipment_feedback.rating` | ❌ Not documented | **Needs clarification** |

---

## 4. Status Value Remapping

| Our UI State | Scroll Engine value | Pidge value | Action |
|---|---|---|---|
| No driver assigned | *(empty `assigned_to`)* | `CREATED` | Map `CREATED` → no-driver UI |
| Rider heading to store | *(not a state)* | `OUT_FOR_PICKUP` | **New state — UI decision needed** |
| Order picked up | `PICKED_UP` | `PICKED_UP` | Direct match |
| Out for delivery | `OUT_FOR_DELIVERY` | `OUT_FOR_DELIVERY` | Direct match |
| Delivered | `DELIVERED` | `DELIVERED` | Direct match |
| Partial delivered | `PARTIAL_DELIVERED` | ❌ Not documented | **Needs clarification** |
| Cancelled | `CANCELLED` | ❌ Not documented | **Needs clarification** |

> **New state `OUT_FOR_PICKUP`:** Rider is en route to the store. We currently have no UI for this.
> Decision needed: Show map with rider moving toward store? Show "Preparing your order" status message?

---

## 5. Status Label Text — We Must Generate

Scroll Engine returned human-readable strings. Pidge does not. We need to define and generate these ourselves based on the status enum.

| Pidge Status | `status_text.order` equivalent | `status_text.driver` equivalent | ETA label |
|---|---|---|---|
| `CREATED` | "Order Confirmed" | "A rider will be assigned soon" | — |
| `OUT_FOR_PICKUP` | "Rider Heading to Store" | "Rider is on the way to pick up your order" | Pickup ETA |
| `PICKED_UP` | "Order Picked Up" | "Rider has collected your order" | — |
| `OUT_FOR_DELIVERY` | "Out for Delivery" | "Rider is on the way to you" | Drop ETA |
| `DELIVERED` | "Order Delivered" | — | — |

> These are suggestions — confirm with product/design before implementing.

---

## 6. Tracking Screen — What Needs to Change

### Polling → Webhook
- Remove 8-second `delay(8000)` + `callShipmentTracking()` loop
- Add backend webhook receiver endpoint
- Backend pushes status updates to app (via FCM or our own push mechanism)
- Keep `/fulfillment/tracking` poll as fallback for live driver GPS when `OUT_FOR_DELIVERY`

### Driver Section
- `assigned_to.first_name + last_name` → `rider.name` (single string)
- `assigned_to.phone` → `rider.mobile`
- Remove third-party delivery check (`assigned_to.email == "hello@home-run.co"`)

### Status Labels
- Remove dependency on `status_text.*` fields
- Add local string mapping from Pidge status enum

### ETA
- `status_text.eta` (string) → format `fulfillment.drop.eta` (ISO UTC) to `Asia/Kolkata` readable string

### Delivery Timestamp
- `history_timeline[i].updatedAt` → `fulfillment.drop.timestamp` from webhook

### Locations
- Store lat/lng: `shipment.start_location` → `fulfillment.pickup.location` (webhook)
- Customer lat/lng: `shipment.end_location` → `fulfillment.drop.location` (webhook)

### New `OUT_FOR_PICKUP` State
- Define UI for rider heading to store (currently not handled)

### Auth
- Add token login + storage on backend
- Pass Bearer token with every Pidge API call
- Handle token expiry / re-login

---

## 7. What Stays the Same

- Map rendering logic (Bezier curves, Google Directions, markers)
- Camera bounds and padding
- Payment flow (CashFree) — Pidge does not handle payments
- Driver feedback UI — pending clarification on Pidge support
- `OrderSummary` navigation — passes same data
- WhatsApp support chat

---

## 8. Open Questions for Pidge

### Must-answer before development:
1. **Order ID linking** — How do we map Pidge order ID back to our Shopify order ID? Is `reference_id` the field we control at order creation?
2. **`track_code`** — Is this a full public URL or just a code? If a code, what is the full URL format?
3. **`PARTIAL_DELIVERED`** — Do you support partial deliveries? What status is sent?
4. **`CANCELLED`** — What is the exact status string when an order is cancelled?
5. **`OUT_FOR_PICKUP` UI** — What data is available during this state — rider GPS, ETA to pickup?
6. **Webhook reliability** — Should we keep polling as a fallback? What is the retry guarantee?
7. **Driver feedback** — Do you have an API to submit and retrieve driver ratings?
8. **Multi-shipment** — Can one Shopify order map to multiple Pidge orders? If yes, how are they linked?
9. **Status label strings** — Can you provide human-readable label strings per status, or do we define them ourselves?
10. **Token TTL** — How long does the auth token last? Should we manage it on the backend only?

### Nice to have:
11. **AWB number** — Is there a tracking/AWB number field in your response?
12. **Rider photo** — Is a rider profile photo URL available?
13. **Sandbox** — Is there a staging environment for development testing?
14. **Webhook auth** — How do we verify incoming webhooks are genuinely from Pidge?
