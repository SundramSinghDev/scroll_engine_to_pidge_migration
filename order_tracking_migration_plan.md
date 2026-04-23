# Order Tracking — Migration Brief
> Purpose: Document the existing Scroll Engine integration so the new service provider
> can map their API contract to our UI requirements.

---

## 1. Current Provider
**Scroll Engine** — `https://api.scrollengine-delivery.com`

---

## 2. Tracking Request Payloads (What We Send)

### 2a. trackOrder (initial load)
Called once on screen open to get order + shipment summary.
```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "order_id": "1234567890"
}
```
> `order_id` is extracted from Shopify GID: `"gid://shopify/Order/1234567890"` → `"1234567890"`

### 2b. trackShipment (live polling)
Called every 8 seconds for live driver location. Only triggered when order is active (not CANCELLED / PARTIAL_DELIVERED).
```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "shipment_id": "<_id from trackOrder response>"
}
```

### 2c. getFeedback (after delivery)
Called to check if driver feedback was already submitted.
```json
{
  "shop_url": "b607ea-2b.myshopify.com",
  "shipment_id": "<_id from trackOrder response>"
}
```

---

## 3. trackOrder Response — Required Keys

```
Response: { status: Boolean, data: Array }
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false triggers retry (max 15 attempts) |
| `data[].shipment._id` | String | Yes | Primary shipment identifier — used in all subsequent calls |
| `data[].shipment.shipment_status` | String | Yes | Controls entire UI state (see Status Values section) |
| `data[].shipment.awb` | String | Optional | Shown in multi-shipment tab labels |
| `data[].shipment.start_location.order_location.lat` | String | Yes | Store/pickup lat — map marker + camera bounds |
| `data[].shipment.start_location.order_location.lng` | String | Yes | Store/pickup lng — map marker + camera bounds |
| `data[].shipment.end_location.order_location.lat` | String | Yes | Customer delivery lat — map marker + camera bounds |
| `data[].shipment.end_location.order_location.lng` | String | Yes | Customer delivery lng — map marker + camera bounds |
| `data[].shipment.trackingInfo.url` | String | Yes | Shareable public tracking URL shown to user |
| `data[].status_text.order_confirmation_msg` | String | Optional | Shown below status header. "Your order will get processed" → centred 14sp. Else → 12sp left, timer icon shown |
| `data[].payment.payment_link` | String | Optional | Shareable payment URL (shown when outstanding amount > 0) |
| `data[].payment.outstanding_amount` | String (Double) | Optional | > 0 → shows Pay Now button. 0 or absent → shows Paid label |
| `data[0].order.name` | String | Optional | Order display name (e.g. `#1234`) used in WhatsApp and share messages |

---

## 4. trackShipment Response — Required Keys

```
Response: { status: Boolean, data: Array }
All data read from data[0]
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false exits silently |
| `data[0].all_shipments_info` | JSONArray | Optional | Builds multi-shipment tabs. Each entry needs `_id`, `awb`, `shipment_status` |
| `data[0].shipment._id` | String | Yes | Updates active `shipmentId` |
| `data[0].shipment.shipment_status` | String | Yes | Core UI branch (see Status Values section) |
| `data[0].shipment.start_location.order_location.lat/lng` | String | Yes | Refreshes store map marker |
| `data[0].shipment.end_location.order_location.lat/lng` | String | Yes | Refreshes delivery map marker |
| `data[0].shipment.trackingInfo.url` | String | Yes | Refreshes shareable tracking link |
| `data[0].assigned_to` | JSONObject | Yes | Driver info. Empty object (`{}`) = no driver assigned yet |
| `data[0].assigned_to.email` | String | Yes | `"hello@home-run.co"` = third-party delivery → hide map/call UI |
| `data[0].assigned_to.first_name` | String | Yes | Driver name shown in UI |
| `data[0].assigned_to.last_name` | String | Yes | Driver name shown in UI |
| `data[0].assigned_to.phone` | String | Yes | Driver phone — wired to call button (`tel:` intent) |
| `data[0].status_text.order` | String | Yes | Main status label (e.g. "Out for delivery", "Order picked up") |
| `data[0].status_text.order_confirmation_msg` | String | Optional | Sub-label under status header |
| `data[0].status_text.driver` | String | Yes | Driver status message shown in driver section |
| `data[0].status_text.eta` | String | Yes (when OUT_FOR_DELIVERY) | ETA string — overrides `order_confirmation_msg` when driver is out for delivery |
| `data[0].driver_geo_location.latitude` | Double | Yes (when driver assigned) | Driver live GPS lat — truck marker on map |
| `data[0].driver_geo_location.longitude` | Double | Yes (when driver assigned) | Driver live GPS lng — truck marker on map |
| `data[0].payment.payment_link` | String | Optional | Same as trackOrder |
| `data[0].payment.outstanding_amount` | String (Double) | Optional | Same as trackOrder |
| `data[0].history_timeline` | JSONArray | Yes (for delivered orders) | Used to find delivery timestamp |
| `data[0].history_timeline[i].status` | String | Yes | Matched against `"DELIVERED"` / `"PARTIAL_DELIVERED"` to find delivery event |
| `data[0].history_timeline[i].updatedAt` | String (ISO UTC) | Yes | Delivery timestamp — converted to `Asia/Kolkata`, shown as `"Your order was delivered HH:MM AM/PM"` |

---

## 5. getFeedback Response — Required Keys

```
Response: { status: Boolean, data: Array }
```

| Key Path | Type | Required | UI Use Case |
|---|---|---|---|
| `status` | Boolean | Yes | Gate — false hides entire feedback section |
| `data[0].shipment_feedback` | JSONObject | Yes | Feedback container |
| `data[0].shipment_feedback.rating` | String (Int) | Optional | **Present** → lock rating bar at this value (read-only, shows "already submitted" icon). **Absent** → unlock rating bar for new submission |

---

## 6. Shipment Status Values — Full UI Matrix

| Status | Map | Status Header | Driver Section | Polling |
|---|---|---|---|---|
| `OUT_FOR_DELIVERY` | Truck marker at driver GPS + Google Directions route (red line) | Yellow | Shown + call button enabled, ETA displayed | Every 8s |
| `PICKED_UP` | Store + delivery markers + dashed Bezier curve | Yellow | Shown + call button enabled | Every 8s |
| `DELIVERED` | Hidden | Green, "Order Delivered" | Hidden | Stops |
| `PARTIAL_DELIVERED` | Hidden | Green, "Order Delivered" | Hidden | Stops |
| `CANCELLED` | Visible but cleared | Red, "Order Cancelled" | Hidden | Stops |
| *(no driver assigned)* | Store + delivery markers + dashed Bezier curve | Yellow | Hidden (no call/message) | Every 8s |

---

## 7. Multi-Shipment Support

When an order has multiple shipments:
- `all_shipments_info` (from `trackShipment`) or synthetic tabs built from `trackOrder.data[]` entries
- Each tab entry needs: `_id`, `awb`, `shipment_status`
- Switching tabs re-calls `trackShipment` with the selected shipment's `_id`
- `selectedDeliveryIndex` tracks active tab to prevent stale responses resetting the view

---

## 8. Third-Party Delivery Detection

When `assigned_to.email == "hello@home-run.co"`:
- Map div hidden
- Driver call button hidden
- Driver message section hidden
- `thirdPartyDeliveryInfo` view shown instead
- Polling continues normally

---

## 9. Payment Flow (within tracking screen)

Triggered when `outstanding_amount > 0`:
1. Pay Now button shown
2. On click → `rePayment(orderId)` → CashFree order created
3. CashFree SDK handles payment
4. On `onPaymentVerify()` callback → `updatePaymentInOrder(orderId, txnId)`
5. Screen refreshes tracking state

---

## 10. Feedback Submission Flow (post-delivery)

1. `getFeedback(shipmentId)` called after delivery confirmed
2. If no existing rating → rating bar enabled
3. User selects stars → `DriverFeedBackBottomSheet` opens (with `shipmentId` + rating)
4. On success → `getFeedback()` re-called → bar locked + "already submitted" icon shown
5. `FeedBackSubmittedBottomSheet` shown with result

---

## 11. Questions for New Provider

- Do you support a single endpoint for both initial load and live polling, or separate?
- What is the polling strategy — push (WebSocket/SSE) or pull?
- Will `shipment_status` values match the existing enum, or do we need to remap?
- Will driver GPS be included in the same response or a separate endpoint?
- Do you support multi-shipment orders with an `all_shipments_info` array?
- Will you provide a shareable public tracking URL (`trackingInfo.url`)?
- Do you have a delivery history timeline for confirmed delivery timestamps?
- Will you include a payment/outstanding amount field, or is that handled separately?
- How do you identify third-party deliveries?
