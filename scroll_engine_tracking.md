# Scroll Engine — Existing Order Tracking Integration

> Current provider documentation extracted from `OrderStatusPage.kt`, `OrderDetailsViewModel`, and related files.

---

## 1. Provider Details

- **Provider:** Scroll Engine
- **Base URL:** `https://api.scrollengine-delivery.com`
- **Auth:** None — requests use `shop_url` as identifier
- **Polling strategy:** Pull — app polls every 8 seconds

---

## 2. Request Payloads

### 2a. trackOrder
Called once on screen open. Retried up to 15 times if `status = false`.

```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "order_id": "1234567890"
}
```
> `order_id` extracted from Shopify GID: `"gid://shopify/Order/1234567890"` → `"1234567890"`

---

### 2b. trackShipment
Called every 8 seconds when order is active. Not called for CANCELLED or PARTIAL_DELIVERED.

```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "shipment_id": "<data[].shipment._id from trackOrder>"
}
```

---

### 2c. getFeedback
Called after delivery is confirmed to check if driver feedback already exists.

```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "shipment_id": "<data[].shipment._id from trackOrder>"
}
```

---

## 3. trackOrder — Response Structure

```
{ status: Boolean, data: Array }
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false triggers retry (max 15 attempts), then `onBackPressed()` |
| `data[].shipment._id` | String | Yes | Primary shipment ID — used in all subsequent API calls |
| `data[].shipment.shipment_status` | String | Yes | Controls entire UI state (see Section 6) |
| `data[].shipment.awb` | String | Optional | Shown in multi-shipment tab label |
| `data[].shipment.start_location.order_location.lat` | String | Yes | Store/pickup latitude — map marker + camera bounds |
| `data[].shipment.start_location.order_location.lng` | String | Yes | Store/pickup longitude — map marker + camera bounds |
| `data[].shipment.end_location.order_location.lat` | String | Yes | Customer delivery latitude — map marker + camera bounds |
| `data[].shipment.end_location.order_location.lng` | String | Yes | Customer delivery longitude — map marker + camera bounds |
| `data[].shipment.trackingInfo.url` | String | Yes | Shareable public tracking page URL |
| `data[].status_text.order_confirmation_msg` | String | Optional | Shown below status header. "Your order will get processed" → centred 14sp, timer icon hidden. Else → 12sp left-aligned, timer icon shown |
| `data[].payment.payment_link` | String | Optional | Shareable payment URL |
| `data[].payment.outstanding_amount` | String (Double) | Optional | > 0 → shows Pay Now button. 0 or absent → shows Paid label |
| `data[0].order.name` | String | Optional | Shopify order display name e.g. `#1234` — used in WhatsApp message and share text |

---

## 4. trackShipment — Response Structure

```
{ status: Boolean, data: Array }
All data read from data[0]
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false exits silently |
| `data[0].all_shipments_info` | JSONArray | Optional | Builds multi-shipment tab bar. Each entry needs `_id`, `awb`, `shipment_status` |
| `data[0].shipment._id` | String | Yes | Updates active `shipmentId` |
| `data[0].shipment.shipment_status` | String | Yes | Core UI branch control |
| `data[0].shipment.start_location.order_location.lat/lng` | String | Yes | Refreshes store map marker position |
| `data[0].shipment.end_location.order_location.lat/lng` | String | Yes | Refreshes delivery map marker position |
| `data[0].shipment.trackingInfo.url` | String | Yes | Refreshes shareable tracking link |
| `data[0].assigned_to` | JSONObject | Yes | Driver block. Empty object = no driver assigned yet |
| `data[0].assigned_to.email` | String | Yes | `"hello@home-run.co"` = third-party delivery → hide map/call UI, show `thirdPartyDeliveryInfo` |
| `data[0].assigned_to.first_name` | String | Yes | Driver first name |
| `data[0].assigned_to.last_name` | String | Yes | Driver last name — concatenated with first_name for display |
| `data[0].assigned_to.phone` | String | Yes | Driver phone — wired to `tel:` call intent |
| `data[0].status_text.order` | String | Yes | Main status label text (e.g. "Out for delivery", "Order picked up") |
| `data[0].status_text.order_confirmation_msg` | String | Optional | Sub-label below status header |
| `data[0].status_text.driver` | String | Yes | Driver status message in driver info section |
| `data[0].status_text.eta` | String | Yes (OUT_FOR_DELIVERY) | ETA string — overrides `order_confirmation_msg` when OUT_FOR_DELIVERY |
| `data[0].driver_geo_location.latitude` | Double | Yes (driver assigned) | Driver live GPS latitude — truck marker on map |
| `data[0].driver_geo_location.longitude` | Double | Yes (driver assigned) | Driver live GPS longitude — truck marker on map |
| `data[0].payment.payment_link` | String | Optional | Same as trackOrder |
| `data[0].payment.outstanding_amount` | String (Double) | Optional | Same as trackOrder |
| `data[0].history_timeline` | JSONArray | Yes (delivered) | Delivery timestamp lookup |
| `data[0].history_timeline[i].status` | String | Yes | Matched against `"DELIVERED"` / `"PARTIAL_DELIVERED"` |
| `data[0].history_timeline[i].updatedAt` | String (ISO UTC) | Yes | Delivery timestamp → converted to Asia/Kolkata → shown as `"Your order was delivered HH:MM AM/PM"` |

---

## 5. getFeedback — Response Structure

```
{ status: Boolean, data: Array }
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false hides entire feedback section |
| `data[0].shipment_feedback` | JSONObject | Yes | Feedback container |
| `data[0].shipment_feedback.rating` | String (Int) | Optional | **Present** → lock rating bar (read-only), show "already submitted" icon. **Absent** → unlock bar for new submission |

---

## 6. Shipment Status Values — Full UI Matrix

| Status | Map | Status Header Colour | Driver Section | Next Action |
|---|---|---|---|---|
| `OUT_FOR_DELIVERY` | Truck marker at driver GPS + red Google Directions route | Yellow | Shown, call enabled, ETA displayed | Poll every 8s |
| `PICKED_UP` | Store + delivery markers + dashed Bezier curve | Yellow | Shown, call enabled | Poll every 8s |
| `DELIVERED` | Hidden | Green — "Order Delivered" | Hidden | Stop polling, show feedback |
| `PARTIAL_DELIVERED` | Hidden | Green — "Order Delivered" | Hidden | Stop polling, show feedback |
| `CANCELLED` | Visible but cleared | Red — "Order Cancelled" | Hidden | Stop polling |
| *(no driver assigned)* | Store + delivery markers + dashed Bezier curve | Yellow | Hidden (no call/message) | Poll every 8s |

---

## 7. Multi-Shipment Support

- `all_shipments_info[]` array from `trackShipment` (or synthetic tabs built from `trackOrder.data[]`)
- Each tab entry needs: `_id`, `awb`, `shipment_status`
- Tab switch → re-calls `trackShipment` with selected shipment `_id`
- `selectedDeliveryIndex` guards against stale responses resetting active tab

---

## 8. Third-Party Delivery

Detected when `assigned_to.email == "hello@home-run.co"`:
- Map hidden
- Driver call button hidden
- Driver message section hidden
- `thirdPartyDeliveryInfo` view shown
- Polling continues normally

---

## 9. Payment Flow

Triggered when `payment.outstanding_amount > 0`:
1. Pay Now button visible
2. User taps → `rePayment(orderId)` → CashFree order created
3. CashFree SDK handles payment UI
4. `onPaymentVerify()` callback → `updatePaymentInOrder(orderId, txnId)`
5. Screen re-fetches tracking state

---

## 10. Driver Feedback Flow

1. `getFeedback(shipmentId)` called after DELIVERED/PARTIAL_DELIVERED confirmed
2. No existing rating → rating bar enabled, `ic_custom_delivery_illustration` shown
3. User selects stars → `DriverFeedBackBottomSheet` opens (shipmentId + rating passed)
4. Success → `getFeedback()` re-called → bar locked, `ic_feedback_already_added` shown
5. `FeedBackSubmittedBottomSheet` shown with result
6. Failure → rating cleared, `FeedBackSubmittedBottomSheet` shown with failure state

---

## 11. Map Rendering Details

| Scenario | Map Contents |
|---|---|
| Order active, no driver | Store marker + Delivery marker + dashed black Bezier curve |
| PICKED_UP | Store marker + Delivery marker + dashed black Bezier curve |
| OUT_FOR_DELIVERY | Delivery marker + Driver truck marker + red route line (Google Directions API) |
| DELIVERED / CANCELLED | Map hidden |

- **Bezier curve:** Quadratic curve with depth factor `0.2`, rendered as dashed polyline (Dash 20f / Gap 10f), width 5f, black
- **Google Directions route:** `https://maps.googleapis.com/maps/api/directions/xml` — called on background thread, result parsed and drawn as red polyline (width 3f)
- Camera bounds padding: 100px
