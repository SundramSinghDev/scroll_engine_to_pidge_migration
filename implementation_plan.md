# Pidge Migration — Implementation Plan

## Context
Migrating order tracking from Scroll Engine to Pidge. Decisions confirmed:
- Pidge order ID stored as Shopify order **metafield** (`namespace: "custom"`, key TBD with Pidge)
- Auth token managed **by the app** — stored in MagePrefs, refreshed on expiry
- **Both webhook + polling** implemented, switched via `Constant.PIDGE_USE_WEBHOOK` flag

---

## Implementation Status

| # | File | Change | Status |
|---|---|---|---|
| 1 | `utils/Constant.kt` | Add Pidge metafield alias/key constants + webhook flag | ✅ Done |
| 2 | `sharedprefsection/MagePrefs.kt` | Add Pidge token + expiry save/get methods | ✅ Done |
| 3 | `utils/Urls.kt` | Add Pidge base URL + endpoint constants | ✅ Done |
| 4 | `utils/ApiCallInterface.kt` | Add Pidge login, order status, rider tracking endpoints | ✅ Done |
| 5 | `repositories/Repository.kt` | Add thin wrapper methods for Pidge calls | ✅ Done |
| 6 | `shopifyqueries/Query.kt` | Add Pidge order ID metafield to order queries | ✅ Done |
| 7 | `ordersection/viewmodels/OrderDetailsViewModel.kt` | Add Pidge LiveData + auth + tracking methods | ✅ Done |
| 8 | `ordersection/activities/OrderStatusPage.kt` | Replace observers, response handling, add mode switch | 🔄 In Progress |

---

## Step 1 — `utils/Constant.kt` ✅

Added after `DELIVERY_FEEDBACK_KEY`:

```kotlin
// Pidge
const val PIDGE_ORDER_ID_KEY = "pidge_order_id"
const val PIDGE_ORDER_ID_ALIAS = "pidge_order_id_alias"
const val PIDGE_USE_WEBHOOK = false  // false = polling, true = webhook-driven
```

---

## Step 2 — `sharedprefsection/MagePrefs.kt` ✅

Added Pidge auth token methods at end of object:

```kotlin
fun savePidgeToken(token: String)
fun getPidgeToken(): String?
fun savePidgeTokenExpiry(expiryMs: Long)
fun getPidgeTokenExpiry(): Long
fun clearPidgeToken()
fun isPidgeTokenValid(): Boolean   // checks token non-null AND not expired
```

Token TTL: `System.currentTimeMillis() + 23 * 60 * 60 * 1000L` (23 hours) on save.

---

## Step 3 — `utils/Urls.kt` ✅

Added before `/*SCROLL ENGINE*/` block:

```kotlin
/*PIDGE*/
private const val PIDGE_BASE_URL = "https://api.pidge.in"   // confirm with Pidge
const val PidgeLogin: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/login"
const val PidgeGetOrder: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}"
const val PidgeGetRiderTracking: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}/fulfillment/tracking"
const val PIDGE_USERNAME = ""   // fill in with Pidge credentials
const val PIDGE_PASSWORD = ""   // fill in with Pidge credentials
```

> **TODO before production:** fill in `PIDGE_USERNAME` and `PIDGE_PASSWORD`. Confirm exact base URL with Pidge.

---

## Step 4 — `utils/ApiCallInterface.kt` ✅

Added after `trackShipment`:

```kotlin
/*PIDGE START*/
@POST(Urls.PidgeLogin)
fun pidgeLogin(@Body params: JsonObject): Single<JsonElement>

@GET(Urls.PidgeGetOrder)
fun pidgeGetOrder(
    @Header("Authorization") token: String,
    @Path("id") orderId: String
): Single<JsonElement>

@GET(Urls.PidgeGetRiderTracking)
fun pidgeGetRiderTracking(
    @Header("Authorization") token: String,
    @Path("id") orderId: String
): Single<JsonElement>
/*PIDGE END*/
```

---

## Step 5 — `repositories/Repository.kt` ✅

Added thin wrappers after `trackShipment()`:

```kotlin
fun pidgeLogin(params: JsonObject): Single<JsonElement>
fun pidgeGetOrder(token: String, orderId: String): Single<JsonElement>
fun pidgeGetRiderTracking(token: String, orderId: String): Single<JsonElement>
```

---

## Step 6 — `shopifyqueries/Query.kt` ✅

Added Pidge metafield after `DELIVERY_FEEDBACK_ALIAS` in the order query builder chain:

```kotlin
}.withAlias(Constant.PIDGE_ORDER_ID_ALIAS)
.metafield(Constant.PIDGE_ORDER_ID_KEY,
    { s -> s.namespace(Constant.CUSTOM_META_FIELD_NAME_SPACE) }) { m ->
    m.key().value().namespace()
}
```

---

## Step 7 — `ordersection/viewmodels/OrderDetailsViewModel.kt` ✅

### 7a. New LiveData fields (added near top):
```kotlin
val pidgeOrderTracking: MutableLiveData<ApiResponse> = MutableLiveData()
val pidgeShipmentTracking: MutableLiveData<ApiResponse> = MutableLiveData()
private val _pidgeLoginResponse: MutableLiveData<ApiResponse> = MutableLiveData()
val pidgeLoginResponse: LiveData<ApiResponse> = _pidgeLoginResponse
```

### 7b–7d. New methods added after `trackShipment()`:
- `pidgeLogin()` — builds auth body, calls API, parses token, saves to MagePrefs, posts to `_pidgeLoginResponse`
- `fetchPidgeOrder(pidgeOrderId: String)` — calls `pidgeGetOrder`, posts result to `pidgeOrderTracking`
- `fetchPidgeRiderTracking(pidgeOrderId: String)` — calls `pidgeGetRiderTracking`, posts result to `pidgeShipmentTracking`

---

## Step 8 — `ordersection/activities/OrderStatusPage.kt` 🔄 In Progress

### What is done:
- Added `pidgeOrderId: String?` and `pidgeBackgroundApiCall: Job?` fields
- Added Pidge observers (`pidgeLoginResponse`, `pidgeOrderTracking`, `pidgeShipmentTracking`) in `onCreate`
- Added `pidgeOrderId` extraction from metafield after jsonobject setup
- Replaced the two initial `model?.trackOrder(jsonobject)` entry point calls (CANCELLED + normal track branches) with `startTracking()`

### What remains (continuing from where interrupted):
Add the following private functions to the activity class:

#### `startTracking()` — entry point
```kotlin
private fun startTracking() {
    val pidgeId = pidgeOrderId
    if (!pidgeId.isNullOrEmpty()) {
        ensurePidgeTokenThenTrack(pidgeId)
    } else {
        model?.trackOrder(jsonobject)  // Scroll Engine fallback
    }
}
```

#### `ensurePidgeTokenThenTrack()`
```kotlin
private fun ensurePidgeTokenThenTrack(pidgeId: String) {
    if (MagePrefs.isPidgeTokenValid()) {
        model?.fetchPidgeOrder(pidgeId)
    } else {
        model?.pidgeLogin()
        // pidgeLoginResponse observer calls fetchPidgeOrder on success
    }
}
```

#### `showPidgeOrderStatus()` — replaces `showMap()` for Pidge orders
Maps Pidge `GET /order/:id` response. Response shape:
```json
{ id, status, fulfillment: { status, pickup: { location }, drop: { location, timestamp }, rider, logs[] } }
```
- Sets status header text and background colour
- Extracts store lat/lng (pickup.location) and user lat/lng (drop.location)
- Sets rider name + call button
- Calls `manageTimerLabelForDeliveredOrderFromPidge()` if DELIVERED
- Calls `startPidgePolling()` if not DELIVERED/CANCELLED and `!Constant.PIDGE_USE_WEBHOOK`
- Calls `getFeedBack("showPidgeOrderStatus")`

#### `showPidgeRiderLocation()` — replaces `showDriverLocation()` for Pidge orders
Maps `GET /order/:id/fulfillment/tracking` response:
```json
{ data: { rider: { name, mobile }, status, location: { latitude, longitude } } }
```
- Updates driver name + call button
- Clears map and draws markers/polylines based on status
- `OUT_FOR_DELIVERY`: truck icon + Google Directions route
- Other statuses: store/user markers + Bezier polyline
- Schedules next 8s poll via `pidgeBackgroundApiCall` if polling mode and not DELIVERED/CANCELLED

#### `startPidgePolling()`
```kotlin
private fun startPidgePolling() {
    pidgeBackgroundApiCall?.cancel()
    pidgeOrderId?.let { model?.fetchPidgeRiderTracking(it) }
}
```

#### `getPidgeStatusLabel(status: String): String`
```kotlin
"CREATED"          -> "Order Confirmed"
"OUT_FOR_PICKUP"   -> "Rider Heading to Store"
"PICKED_UP"        -> "Order Picked Up"
"OUT_FOR_DELIVERY" -> "Out for Delivery"
"DELIVERED"        -> "Order Delivered"
"CANCELLED"        -> "Order Cancelled"
else               -> "Tracking your order"
```

#### `getPidgeStatusBackground(status: String): Int`
```kotlin
"DELIVERED" -> R.drawable.gradiantgreen
"CANCELLED" -> R.drawable.gradiantred
else        -> R.drawable.gradiantyellow
```

#### `manageTimerLabelForDeliveredOrderFromPidge(fulfillment: JSONObject)`
- Sets `isDeliveredMarkedFromScrollEngine = true`
- Hides map fragment, shows `statusicn`
- Hides call/message/feedback sections, shows `addFeedback`
- Reads `drop.timestamp` for delivery time text

#### `onDestroy()` override
```kotlin
override fun onDestroy() {
    pidgeBackgroundApiCall?.cancel()
    super.onDestroy()
}
```

#### Webhook mode hint in `onNewIntent()` / `onResume()`
When `PIDGE_USE_WEBHOOK = true`, check intent for `"pidge_update"` flag → call `startTracking()`.

---

## Status Mapping Reference

| Pidge Status | Header Text | Header Colour | Map | Polling |
|---|---|---|---|---|
| `CREATED` | "Order Confirmed" | Yellow | Store + delivery markers + Bezier | Yes |
| `OUT_FOR_PICKUP` | "Rider Heading to Store" | Yellow | Store + delivery markers + Bezier | Yes |
| `PICKED_UP` | "Order Picked Up" | Yellow | Store + delivery markers + Bezier | Yes |
| `OUT_FOR_DELIVERY` | "Out for Delivery" | Yellow | Truck marker + Google Directions route | Yes |
| `DELIVERED` | "Order Delivered" | Green | Hidden | No |
| `CANCELLED` | "Order Cancelled" | Red | Cleared | No |

---

## Scroll Engine Fallback

If `pidgeOrderId` is null/empty (order not on Pidge or metafield not set):
- `startTracking()` falls through to `model?.trackOrder(jsonobject)` — existing Scroll Engine flow unchanged
- No existing Scroll Engine code removed — full backward compat

---

## What Remains Pending (confirm with Pidge)

- Exact Pidge base URL (currently `https://api.pidge.in` — confirm)
- Fill in `PIDGE_USERNAME` and `PIDGE_PASSWORD` in `Urls.kt`
- Exact metafield key name Pidge will use to store Pidge order ID on Shopify (currently `"pidge_order_id"`)
- Driver feedback API (submit + get rating) for Pidge orders
- `PARTIAL_DELIVERED` and `CANCELLED` exact status strings
- Multi-shipment support

---

## Verification Steps

1. Build `assembleDebug` — no compile errors
2. Open order with `pidge_order_id` metafield set → confirm Pidge flow entered
3. Open order without metafield → confirm Scroll Engine flow unchanged
4. Toggle `Constant.PIDGE_USE_WEBHOOK = true` → confirm polling loop does not start
5. Toggle `Constant.PIDGE_USE_WEBHOOK = false` → confirm 8s polling starts after initial load
6. Simulate expired token (clear MagePrefs) → confirm re-login runs before tracking
7. Confirm map markers render correctly for each status
8. Confirm `manageTimerLabelForDeliveredOrderFromPidge` shows correct delivery time
