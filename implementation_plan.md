 ╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
     │ Pidge Migration — Implementation Plan                                                                                  │
     │                                                                                                                        │
     │ Context                                                                                                                │
     │                                                                                                                        │
     │ Migrating order tracking from Scroll Engine to Pidge. Decisions confirmed:                                             │
     │ - Pidge order ID stored as Shopify order metafield (namespace: "custom", key TBD with Pidge)                           │
     │ - Auth token managed by the app — stored in MagePrefs, refreshed on expiry                                             │
     │ - Both webhook + polling implemented, switched via Constant.PIDGE_USE_WEBHOOK flag                                     │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Files to Modify (in execution order)                                                                                   │
     │                                                                                                                        │
     │ ┌─────┬──────────────────────────────────────────────────┬─────────────────────────────────────────────────────────┐   │
     │ │  #  │                       File                       │                         Change                          │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 1   │ utils/Constant.kt                                │ Add Pidge metafield alias/key constants + webhook flag  │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 2   │ sharedprefsection/MagePrefs.kt                   │ Add Pidge token + expiry save/get methods               │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 3   │ utils/Urls.kt                                    │ Add Pidge base URL + endpoint constants                 │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 4   │ utils/ApiCallInterface.kt                        │ Add Pidge login, order status, rider tracking endpoints │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 5   │ repositories/Repository.kt                       │ Add thin wrapper methods for Pidge calls                │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 6   │ shopifyqueries/Query.kt                          │ Add Pidge order ID metafield to order queries           │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 7   │ ordersection/viewmodels/OrderDetailsViewModel.kt │ Add Pidge LiveData + auth + tracking methods            │   │
     │ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤   │
     │ │ 8   │ ordersection/activities/OrderStatusPage.kt       │ Replace observers, response handling, add mode switch   │   │
     │ └─────┴──────────────────────────────────────────────────┴─────────────────────────────────────────────────────────┘   │
     │                                                                                                                        │
     │ No new files needed.                                                                                                   │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 1 — utils/Constant.kt                                                                                             │
     │                                                                                                                        │
     │ Add after existing alias constants (after line 137 DELIVERY_FEEDBACK_KEY):                                             │
     │                                                                                                                        │
     │ // Pidge                                                                                                               │
     │ const val PIDGE_ORDER_ID_KEY = "pidge_order_id"                                                                        │
     │ const val PIDGE_ORDER_ID_ALIAS = "pidge_order_id_alias"                                                                │
     │ const val PIDGE_USE_WEBHOOK = false  // false = polling, true = webhook-driven                                         │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 2 — sharedprefsection/MagePrefs.kt                                                                                │
     │                                                                                                                        │
     │ Add two new key/value pairs for Pidge auth token and its expiry timestamp:                                             │
     │                                                                                                                        │
     │ fun savePidgeToken(token: String) = prefs.edit().putString("PIDGE_TOKEN", token).apply()                               │
     │ fun getPidgeToken(): String? = prefs.getString("PIDGE_TOKEN", null)                                                    │
     │ fun savePidgeTokenExpiry(expiryMs: Long) = prefs.edit().putLong("PIDGE_TOKEN_EXPIRY", expiryMs).apply()                │
     │ fun getPidgeTokenExpiry(): Long = prefs.getLong("PIDGE_TOKEN_EXPIRY", 0L)                                              │
     │ fun clearPidgeToken() = prefs.edit().remove("PIDGE_TOKEN").remove("PIDGE_TOKEN_EXPIRY").apply()                        │
     │                                                                                                                        │
     │ fun isPidgeTokenValid(): Boolean {                                                                                     │
     │     val token = getPidgeToken()                                                                                        │
     │     val expiry = getPidgeTokenExpiry()                                                                                 │
     │     return !token.isNullOrEmpty() && System.currentTimeMillis() < expiry                                               │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ Token TTL: store System.currentTimeMillis() + 23 * 60 * 60 * 1000 (23 hours) on save.                                  │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 3 — utils/Urls.kt                                                                                                 │
     │                                                                                                                        │
     │ Add inside the Data object (alongside SCROLL_ENGINE_BASE_URL block):                                                   │
     │                                                                                                                        │
     │ // Pidge                                                                                                               │
     │ private const val PIDGE_BASE_URL = "https://api.pidge.in"   // confirm exact base URL with Pidge                       │
     │ const val PidgeLogin: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/login"                                       │
     │ const val PidgeGetOrder: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}"                               │
     │ const val PidgeGetRiderTracking: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}/fulfillment/tracking"  │
     │                                                                                                                        │
     │ // Pidge credentials (move to BuildConfig or secure storage before production)                                         │
     │ const val PIDGE_USERNAME = ""   // fill in                                                                             │
     │ const val PIDGE_PASSWORD = ""   // fill in                                                                             │
     │                                                                                                                        │
     │ Note: Confirm the exact Pidge base URL before filling in. The Postman doc didn't expose it explicitly.                 │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 4 — utils/ApiCallInterface.kt                                                                                     │
     │                                                                                                                        │
     │ Add after the /*SCROLL ENGINE START*/ block:                                                                           │
     │                                                                                                                        │
     │ /*PIDGE START*/                                                                                                        │
     │                                                                                                                        │
     │ @POST(Urls.PidgeLogin)                                                                                                 │
     │ fun pidgeLogin(@Body params: JsonObject): Single<JsonElement>                                                          │
     │                                                                                                                        │
     │ @GET(Urls.PidgeGetOrder)                                                                                               │
     │ fun pidgeGetOrder(                                                                                                     │
     │     @Header("Authorization") token: String,                                                                            │
     │     @Path("id") orderId: String                                                                                        │
     │ ): Single<JsonElement>                                                                                                 │
     │                                                                                                                        │
     │ @GET(Urls.PidgeGetRiderTracking)                                                                                       │
     │ fun pidgeGetRiderTracking(                                                                                             │
     │     @Header("Authorization") token: String,                                                                            │
     │     @Path("id") orderId: String                                                                                        │
     │ ): Single<JsonElement>                                                                                                 │
     │                                                                                                                        │
     │ /*PIDGE END*/                                                                                                          │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 5 — repositories/Repository.kt                                                                                    │
     │                                                                                                                        │
     │ Add thin wrappers after trackShipment():                                                                               │
     │                                                                                                                        │
     │ fun pidgeLogin(params: JsonObject): Single<JsonElement> =                                                              │
     │     apiCallInterface.pidgeLogin(params)                                                                                │
     │                                                                                                                        │
     │ fun pidgeGetOrder(token: String, orderId: String): Single<JsonElement> =                                               │
     │     apiCallInterface.pidgeGetOrder(token, orderId)                                                                     │
     │                                                                                                                        │
     │ fun pidgeGetRiderTracking(token: String, orderId: String): Single<JsonElement> =                                       │
     │     apiCallInterface.pidgeGetRiderTracking(token, orderId)                                                             │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 6 — shopifyqueries/Query.kt                                                                                       │
     │                                                                                                                        │
     │ In the order query (around line 1809 where INVOICE_LINK_ALIAS is defined), add the Pidge metafield:                    │
     │                                                                                                                        │
     │ .withAlias(Constant.PIDGE_ORDER_ID_ALIAS)                                                                              │
     │ .metafield(Constant.PIDGE_ORDER_ID_KEY,                                                                                │
     │     { s -> s.namespace(Constant.CUSTOM_META_FIELD_NAME_SPACE) }) { m ->                                                │
     │     m.key().value().namespace()                                                                                        │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ This must be added in every order query that feeds OrderStatusPage — search for INVOICE_LINK_ALIAS in Query.kt and add │
     │ it right after in the same query builder chain.                                                                        │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 7 — ordersection/viewmodels/OrderDetailsViewModel.kt                                                              │
     │                                                                                                                        │
     │ 7a. New LiveData fields (add near top with other MutableLiveData):                                                     │
     │                                                                                                                        │
     │ val pidgeOrderTracking: MutableLiveData<ApiResponse> = MutableLiveData()                                               │
     │ val pidgeShipmentTracking: MutableLiveData<ApiResponse> = MutableLiveData()                                            │
     │ private val _pidgeLoginResponse: MutableLiveData<ApiResponse> = MutableLiveData()                                      │
     │ val pidgeLoginResponse: LiveData<ApiResponse> = _pidgeLoginResponse                                                    │
     │                                                                                                                        │
     │ 7b. Pidge login method:                                                                                                │
     │                                                                                                                        │
     │ fun pidgeLogin() {                                                                                                     │
     │     val params = JsonObject().apply {                                                                                  │
     │         addProperty("username", Urls.PIDGE_USERNAME)                                                                   │
     │         addProperty("password", Urls.PIDGE_PASSWORD)                                                                   │
     │     }                                                                                                                  │
     │     doRetrofitCall(                                                                                                    │
     │         repository.pidgeLogin(params),                                                                                 │
     │         disposables,                                                                                                   │
     │         customResponse = object : CustomResponse {                                                                     │
     │             override fun onSuccessRetrofit(result: JsonElement) {                                                      │
     │                 try {                                                                                                  │
     │                     val token = result.asJsonObject                                                                    │
     │                         .getAsJsonObject("data")                                                                       │
     │                         .get("token").asString  // "Bearer eyJ..."                                                     │
     │                     MagePrefs.savePidgeToken(token)                                                                    │
     │                     MagePrefs.savePidgeTokenExpiry(                                                                    │
     │                         System.currentTimeMillis() + 23 * 60 * 60 * 1000L                                              │
     │                     )                                                                                                  │
     │                     _pidgeLoginResponse.value = ApiResponse.success(result)                                            │
     │                 } catch (ex: Exception) {                                                                              │
     │                     _pidgeLoginResponse.value = ApiResponse.error(ex)                                                  │
     │                 }                                                                                                      │
     │             }                                                                                                          │
     │             override fun onErrorRetrofit(error: Throwable) {                                                           │
     │                 _pidgeLoginResponse.value = ApiResponse.error(error)                                                   │
     │             }                                                                                                          │
     │         },                                                                                                             │
     │         context = context                                                                                              │
     │     )                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 7c. Pidge order status (replaces trackOrder):                                                                          │
     │                                                                                                                        │
     │ fun fetchPidgeOrder(pidgeOrderId: String) {                                                                            │
     │     val token = MagePrefs.getPidgeToken() ?: return                                                                    │
     │     doRetrofitCall(                                                                                                    │
     │         repository.pidgeGetOrder(token, pidgeOrderId),                                                                 │
     │         disposables,                                                                                                   │
     │         customResponse = object : CustomResponse {                                                                     │
     │             override fun onSuccessRetrofit(result: JsonElement) {                                                      │
     │                 pidgeOrderTracking.value = ApiResponse.success(result)                                                 │
     │             }                                                                                                          │
     │             override fun onErrorRetrofit(error: Throwable) {                                                           │
     │                 pidgeOrderTracking.value = ApiResponse.error(error)                                                    │
     │             }                                                                                                          │
     │         },                                                                                                             │
     │         context = context                                                                                              │
     │     )                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 7d. Pidge rider tracking (replaces trackShipment):                                                                     │
     │                                                                                                                        │
     │ fun fetchPidgeRiderTracking(pidgeOrderId: String) {                                                                    │
     │     val token = MagePrefs.getPidgeToken() ?: return                                                                    │
     │     doRetrofitCall(                                                                                                    │
     │         repository.pidgeGetRiderTracking(token, pidgeOrderId),                                                         │
     │         disposables,                                                                                                   │
     │         customResponse = object : CustomResponse {                                                                     │
     │             override fun onSuccessRetrofit(result: JsonElement) {                                                      │
     │                 pidgeShipmentTracking.value = ApiResponse.success(result)                                              │
     │             }                                                                                                          │
     │             override fun onErrorRetrofit(error: Throwable) {                                                           │
     │                 pidgeShipmentTracking.value = ApiResponse.error(error)                                                 │
     │             }                                                                                                          │
     │         },                                                                                                             │
     │         context = context                                                                                              │
     │     )                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Step 8 — ordersection/activities/OrderStatusPage.kt                                                                    │
     │                                                                                                                        │
     │ This is the largest change. All changes are additive alongside existing Scroll Engine code — the switch is controlled  │
     │ by reading the Pidge metafield.                                                                                        │
     │                                                                                                                        │
     │ 8a. New fields (add near top with existing fields):                                                                    │
     │                                                                                                                        │
     │ private var pidgeOrderId: String? = null                                                                               │
     │ private var pidgeBackgroundApiCall: Job? = null                                                                        │
     │                                                                                                                        │
     │ 8b. Pidge order ID extraction (in onCreate, after orderEdge is set):                                                   │
     │                                                                                                                        │
     │ pidgeOrderId = orderEdge                                                                                               │
     │     ?.withAlias(Constant.PIDGE_ORDER_ID_ALIAS)                                                                         │
     │     ?.metafield?.value                                                                                                 │
     │     ?.takeIf { it.isNotEmpty() }                                                                                       │
     │                                                                                                                        │
     │ 8c. Tracking entry point (replaces model?.trackOrder(jsonobject) calls):                                               │
     │                                                                                                                        │
     │ Add a helper that decides which provider to use:                                                                       │
     │ private fun startTracking() {                                                                                          │
     │     val pidgeId = pidgeOrderId                                                                                         │
     │     if (!pidgeId.isNullOrEmpty()) {                                                                                    │
     │         // Pidge flow                                                                                                  │
     │         ensurePidgeTokenThenTrack(pidgeId)                                                                             │
     │     } else {                                                                                                           │
     │         // Scroll Engine fallback                                                                                      │
     │         model?.trackOrder(jsonobject)                                                                                  │
     │     }                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ private fun ensurePidgeTokenThenTrack(pidgeId: String) {                                                               │
     │     if (MagePrefs.isPidgeTokenValid()) {                                                                               │
     │         model?.fetchPidgeOrder(pidgeId)                                                                                │
     │     } else {                                                                                                           │
     │         model?.pidgeLogin()                                                                                            │
     │         // pidgeLoginResponse observer calls fetchPidgeOrder on success                                                │
     │     }                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8d. New LiveData observers (add in onCreate alongside existing observers):                                             │
     │                                                                                                                        │
     │ model?.pidgeLoginResponse?.observe(this) { response ->                                                                 │
     │     when (response.status) {                                                                                           │
     │         Status.SUCCESS -> {                                                                                            │
     │             pidgeOrderId?.let { model?.fetchPidgeOrder(it) }                                                           │
     │         }                                                                                                              │
     │         else -> {                                                                                                      │
     │             // Token fetch failed — fall back to Scroll Engine                                                         │
     │             model?.trackOrder(jsonobject)                                                                              │
     │         }                                                                                                              │
     │     }                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ model?.pidgeOrderTracking?.observe(this) { showPidgeOrderStatus(it) }                                                  │
     │ model?.pidgeShipmentTracking?.observe(this) { showPidgeRiderLocation(it) }                                             │
     │                                                                                                                        │
     │ 8e. showPidgeOrderStatus() — replaces showMap() for Pidge orders:                                                      │
     │                                                                                                                        │
     │ Maps Pidge GET /order/:id response to existing UI. Response shape:                                                     │
     │ { id, status, fulfillment: { status, pickup: { location }, drop: { location }, rider, drop.eta, logs[] } }             │
     │                                                                                                                        │
     │ private fun showPidgeOrderStatus(response: ApiResponse?) {                                                             │
     │     try {                                                                                                              │
     │         if (response?.data == null) { hideLoader(); return }                                                           │
     │         val data = JSONObject(response.data.toString())                                                                │
     │         val fulfillment = data.optJSONObject("fulfillment") ?: run { hideLoader(); return }                            │
     │         val shipStatus = fulfillment.optString("fulfillment_status", fulfillment.optString("status", ""))              │
     │                                                                                                                        │
     │         // Status header label                                                                                         │
     │         binding.deliveredtxt.text = getPidgeStatusLabel(shipStatus)                                                    │
     │         binding.statussection.setBackgroundResource(getPidgeStatusBackground(shipStatus))                              │
     │                                                                                                                        │
     │         // Locations                                                                                                   │
     │         val pickup = fulfillment.optJSONObject("pickup")                                                               │
     │         val drop = fulfillment.optJSONObject("drop")                                                                   │
     │         storelat = pickup?.optJSONObject("location")?.optString("latitude", "") ?: ""                                  │
     │         storelong = pickup?.optJSONObject("location")?.optString("longitude", "") ?: ""                                │
     │         userlat = drop?.optJSONObject("location")?.optString("latitude", "") ?: ""                                     │
     │         userlong = drop?.optJSONObject("location")?.optString("longitude", "") ?: ""                                   │
     │                                                                                                                        │
     │         // Rider                                                                                                       │
     │         val rider = fulfillment.optJSONObject("rider")                                                                 │
     │         if (rider != null) {                                                                                           │
     │             binding.deliveryName.text = rider.optString("name")                                                        │
     │             binding.assignmancallsect.setOnClickListener {                                                             │
     │                 startActivity(Intent(Intent.ACTION_DIAL).apply {                                                       │
     │                     data = Uri.parse("tel:${rider.optString("mobile")}")                                               │
     │                 })                                                                                                     │
     │             }                                                                                                          │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         // Delivery timestamp from logs (DELIVERED entry)                                                              │
     │         if (shipStatus == "DELIVERED" || shipStatus == "PARTIAL_DELIVERED") {                                          │
     │             manageTimerLabelForDeliveredOrderFromPidge(fulfillment)                                                    │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         // Shipment ID (use Pidge order id as shipment reference for feedback)                                         │
     │         shipmentId = pidgeOrderId                                                                                      │
     │         getFeedBack("showPidgeOrderStatus")                                                                            │
     │                                                                                                                        │
     │         // Trigger rider location fetch based on mode                                                                  │
     │         if (shipStatus != "DELIVERED" && shipStatus != "CANCELLED") {                                                  │
     │             if (!Constant.PIDGE_USE_WEBHOOK) {                                                                         │
     │                 startPidgePolling()                                                                                    │
     │             }                                                                                                          │
     │             // If webhook mode: wait for FCM push — no polling                                                         │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         hideLoader()                                                                                                   │
     │         hideTabSwitchLoader()                                                                                          │
     │     } catch (ex: Exception) {                                                                                          │
     │         exceptionLogFirebase(ex.printStackTrace().toString(), TAG, MagePrefs.getCustomerID())                          │
     │         hideLoader()                                                                                                   │
     │     }                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8f. showPidgeRiderLocation() — replaces showDriverLocation() for Pidge:                                                │
     │                                                                                                                        │
     │ Maps GET /order/:id/fulfillment/tracking response:                                                                     │
     │ { data: { rider: { name, mobile }, status, location: { latitude, longitude } } }                                       │
     │                                                                                                                        │
     │ private fun showPidgeRiderLocation(response: ApiResponse?) {                                                           │
     │     try {                                                                                                              │
     │         if (response?.data == null) { stopRefreshAnimation(); return }                                                 │
     │         val repo = JSONObject(response.data.toString())                                                                │
     │         if (!repo.getBoolean("status")) { stopRefreshAnimation(); return }                                             │
     │         val data = repo.getJSONObject("data")                                                                          │
     │         val shipStatus = data.optString("status", "")                                                                  │
     │         val rider = data.optJSONObject("rider")                                                                        │
     │         val location = data.optJSONObject("location")                                                                  │
     │                                                                                                                        │
     │         shipmentStatus = shipStatus                                                                                    │
     │                                                                                                                        │
     │         // Update driver name and call button                                                                          │
     │         if (rider != null && rider.length() > 0) {                                                                     │
     │             binding.deliveryName.text = rider.optString("name")                                                        │
     │             binding.assignmancallsect.visibility = View.VISIBLE                                                        │
     │             binding.assignmancallsect.setOnClickListener {                                                             │
     │                 startActivity(Intent(Intent.ACTION_DIAL).apply {                                                       │
     │                     data = Uri.parse("tel:${rider.optString("mobile")}")                                               │
     │                 })                                                                                                     │
     │             }                                                                                                          │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         // Map markers + polyline                                                                                      │
     │         if (location != null && isMapInitialized) {                                                                    │
     │             val driverLat = location.optDouble("latitude")                                                             │
     │             val driverLng = location.optDouble("longitude")                                                            │
     │             val driverLatLng = LatLng(driverLat, driverLng)                                                            │
     │             val userLatLng = LatLng(userlat.toDoubleOrNull() ?: 0.0, userlong.toDoubleOrNull() ?: 0.0)                 │
     │             val storeLatLng = LatLng(storelat.toDoubleOrNull() ?: 0.0, storelong.toDoubleOrNull() ?: 0.0)              │
     │                                                                                                                        │
     │             mMap.clear()                                                                                               │
     │             if (shipStatus == "OUT_FOR_DELIVERY") {                                                                    │
     │                 val icon = BitmapDescriptorFactory.fromBitmap(                                                         │
     │                     ResourcesCompat.getDrawable(resources, R.drawable.deliverytruck, null)?.toBitmap()!!               │
     │                 )                                                                                                      │
     │                                                                                                                        │
     │ mMap.addMarker(MarkerOptions().position(userLatLng).title(getString(R.string.delivery_address)).icon(userBitMap))      │
     │                 mMap.addMarker(MarkerOptions().position(driverLatLng).title(rider?.optString("name") ?:                │
     │ "Driver").icon(icon))                                                                                                  │
     │                 binding.reload.visible()                                                                               │
     │                 getDocument(start = driverLatLng, middle = storeLatLng, end = userLatLng, "driving")                   │
     │             } else {                                                                                                   │
     │                 mMap.addMarker(MarkerOptions().position(storeLatLng).title("Store").icon(storeBitMap))                 │
     │                                                                                                                        │
     │ mMap.addMarker(MarkerOptions().position(userLatLng).title(getString(R.string.delivery_address)).icon(userBitMap))      │
     │                 val curvedPoints = getCurvePoints(storeLatLng, userLatLng)                                             │
     │                                                                                                                        │
     │ mMap.addPolyline(PolylineOptions().addAll(curvedPoints).width(5f).color(Color.BLACK).pattern(listOf(Dash(20f),         │
     │ Gap(10f))))                                                                                                            │
     │                 binding.reload.gone()                                                                                  │
     │             }                                                                                                          │
     │             val bounds = LatLngBounds.Builder().include(storeLatLng).include(userLatLng).build()                       │
     │             mMap.animateCamera(CameraUpdateFactory.newLatLngBounds(bounds, 100))                                       │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         // Schedule next poll (polling mode only)                                                                      │
     │         if (!Constant.PIDGE_USE_WEBHOOK && shipStatus != "DELIVERED" && shipStatus != "CANCELLED") {                   │
     │             pidgeBackgroundApiCall = lifecycleScope.launch(Dispatchers.IO) {                                           │
     │                 delay(8000)                                                                                            │
     │                 pidgeOrderId?.let { model?.fetchPidgeRiderTracking(it) }                                               │
     │             }                                                                                                          │
     │         }                                                                                                              │
     │                                                                                                                        │
     │         stopRefreshAnimation()                                                                                         │
     │         hideTabSwitchLoader()                                                                                          │
     │         hideLoader()                                                                                                   │
     │     } catch (ex: Exception) {                                                                                          │
     │         exceptionLogFirebase(ex.printStackTrace().toString(), TAG, MagePrefs.getCustomerID())                          │
     │         stopRefreshAnimation()                                                                                         │
     │     }                                                                                                                  │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8g. Polling starter:                                                                                                   │
     │                                                                                                                        │
     │ private fun startPidgePolling() {                                                                                      │
     │     pidgeBackgroundApiCall?.cancel()                                                                                   │
     │     pidgeOrderId?.let { model?.fetchPidgeRiderTracking(it) }                                                           │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8h. Webhook mode — FCM handler:                                                                                        │
     │                                                                                                                        │
     │ When PIDGE_USE_WEBHOOK = true, the FCM push from our backend should trigger a re-fetch.                                │
     │ In onNewIntent() or onResume(), check for a "pidge_update" flag in the intent and call startTracking().                │
     │ (Backend webhook receiver → FCM → app re-fetches from Pidge directly.)                                                 │
     │                                                                                                                        │
     │ 8i. Status label + colour helpers (add as private functions):                                                          │
     │                                                                                                                        │
     │ private fun getPidgeStatusLabel(status: String): String = when (status) {                                              │
     │     "CREATED"          -> "Order Confirmed"                                                                            │
     │     "OUT_FOR_PICKUP"   -> "Rider Heading to Store"                                                                     │
     │     "PICKED_UP"        -> "Order Picked Up"                                                                            │
     │     "OUT_FOR_DELIVERY" -> "Out for Delivery"                                                                           │
     │     "DELIVERED"        -> "Order Delivered"                                                                            │
     │     "CANCELLED"        -> "Order Cancelled"                                                                            │
     │     else               -> "Tracking your order"                                                                        │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ private fun getPidgeStatusBackground(status: String): Int = when (status) {                                            │
     │     "DELIVERED"        -> R.drawable.gradiantgreen  // confirm drawable name                                           │
     │     "CANCELLED"        -> R.drawable.gradiantred                                                                       │
     │     else               -> R.drawable.gradiantyellow // yellow for in-progress                                          │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8j. Delivery timestamp from Pidge logs:                                                                                │
     │                                                                                                                        │
     │ private fun manageTimerLabelForDeliveredOrderFromPidge(fulfillment: JSONObject) {                                      │
     │     try {                                                                                                              │
     │         isDeliveredMarkedFromScrollEngine = true                                                                       │
     │         supportFragmentManager.beginTransaction().hide(mapFragment).commit()                                           │
     │         binding.statusicn.visibility = View.VISIBLE                                                                    │
     │         binding.deliveredtxt.text = "Order Delivered"                                                                  │
     │         binding.assignmessagesect.gone()                                                                               │
     │         binding.assignmsgsect.gone()                                                                                   │
     │         binding.orderdiv.gone()                                                                                        │
     │         binding.assignmancallsect.gone()                                                                               │
     │         binding.addFeedback.root.visible()                                                                             │
     │         binding.reload.gone()                                                                                          │
     │                                                                                                                        │
     │         // Extract delivery time from drop.timestamp                                                                   │
     │         val dropTimestamp = fulfillment.optJSONObject("drop")?.optString("timestamp", "")                              │
     │         if (!dropTimestamp.isNullOrEmpty()) {                                                                          │
     │             binding.delivedtime.text = "Your order was delivered ${formatTimeStamp(dropTimestamp)}"                    │
     │             binding.delivedtime.visibility = View.VISIBLE                                                              │
     │             binding.delivedtimeSection.visibility = View.VISIBLE                                                       │
     │         }                                                                                                              │
     │     } catch (_: Exception) {}                                                                                          │
     │ }                                                                                                                      │
     │                                                                                                                        │
     │ 8k. Replace all model?.trackOrder(jsonobject) call sites with startTracking():                                         │
     │                                                                                                                        │
     │ Search for model?.trackOrder(jsonobject) in OrderStatusPage.kt — replace each with startTracking().                    │
     │                                                                                                                        │
     │ 8l. Cancel Pidge polling on destroy (add to onDestroy()):                                                              │
     │                                                                                                                        │
     │ pidgeBackgroundApiCall?.cancel()                                                                                       │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Status Mapping Reference                                                                                               │
     │                                                                                                                        │
     │                                                                                                                        │
     │ ┌──────────────────┬──────────────────────────┬───────────────┬────────────────────────────────────────┬─────────┐     │
     │ │   Pidge Status   │       Header Text        │ Header Colour │                  Map                   │ Polling │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ CREATED          │ "Order Confirmed"        │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ OUT_FOR_PICKUP   │ "Rider Heading to Store" │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ PICKED_UP        │ "Order Picked Up"        │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ OUT_FOR_DELIVERY │ "Out for Delivery"       │ Yellow        │ Truck marker + Google Directions route │ Yes     │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ DELIVERED        │ "Order Delivered"        │ Green         │ Hidden                                 │ No      │     │
     │ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤     │
     │ │ CANCELLED        │ "Order Cancelled"        │ Red           │ Cleared                                │ No      │     │
     │ └──────────────────┴──────────────────────────┴───────────────┴────────────────────────────────────────┴─────────┘     │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Scroll Engine Fallback                                                                                                 │
     │                                                                                                                        │
     │ If pidgeOrderId is null/empty (order not yet on Pidge, or metafield not set):                                          │
     │ - startTracking() falls through to model?.trackOrder(jsonobject) → existing Scroll Engine flow unchanged               │
     │ - No existing Scroll Engine code is removed — full backward compatibility                                              │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ What Remains Pending (confirm with Pidge)                                                                              │
     │                                                                                                                        │
     │ - Exact Pidge base URL                                                                                                 │
     │ - Exact metafield key name they will use to store Pidge order ID on Shopify                                            │
     │ - Driver feedback API (submit + get rating)                                                                            │
     │ - PARTIAL_DELIVERED and CANCELLED exact status strings                                                                 │
     │ - track_code — full URL or code only                                                                                   │
     │ - Multi-shipment support                                                                                               │
     │                                                                                                                        │
     │ ---                                                                                                                    │
     │ Verification Steps                                                                                                     │
     │                                                                                                                        │
     │ 1. Build assembleDebug — no compile errors                                                                             │
     │ 2. Open an order that has pidge_order_id metafield set → confirm Pidge flow is entered                                 │
     │ 3. Open an order without the metafield → confirm Scroll Engine flow is unchanged                                       │
     │ 4. Toggle Constant.PIDGE_USE_WEBHOOK = true → confirm polling loop does not start                                      │
     │ 5. Toggle Constant.PIDGE_USE_WEBHOOK = false → confirm 8s polling starts after initial load                            │
     │ 6. Simulate expired token (clear MagePrefs) → confirm re-login runs before tracking                                    │
     │ 7. Confirm map markers render correctly for each status                                                                │
     │ 8. Confirm manageTimerLabelForDeliveredOrderFromPidge shows correct delivery time                                      │
     ╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

● Waiting for your feedback on the plan before we proceed.
                                                                                          
✻ Crunched for 3m 28s                  
                                                                                          
● How is Claude doing this session? (optional)
  1: Bad    2: Fine   3: Good   0: Dismiss    
                                                                                                                               
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────❯                                                                                                                              ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────  ⏵⏵ accept edits on (meta+m to cycle)                                                                     ◐ medium · /effort                                                                                                                                 
 ▐▛███▜▌   Claude Code v2.1.72
▝▜█████▛▘  Sonnet 4.6 · Claude Team
  ▘▘ ▝▝    C:\studio_projects\New folder\bulk_qty_price_patch\14521_home_run_V3

  ↑ Install the PyCharm plugin from the JetBrains Marketplace: https://docs.claude.com/s/claude-code-jetbrains

╭─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│ Plan to implement                                                                                                           │
│                                                                                                                             │
│ Pidge Migration — Implementation Plan                                                                                       │
│                                                                                                                             │
│ Context                                                                                                                     │
│                                                                                                                             │
│ Migrating order tracking from Scroll Engine to Pidge. Decisions confirmed:                                                  │
│ - Pidge order ID stored as Shopify order metafield (namespace: "custom", key TBD with Pidge)                                │
│ - Auth token managed by the app — stored in MagePrefs, refreshed on expiry                                                  │
│ - Both webhook + polling implemented, switched via Constant.PIDGE_USE_WEBHOOK flag                                          │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Files to Modify (in execution order)                                                                                        │
│                                                                                                                             │
│ ┌─────┬──────────────────────────────────────────────────┬─────────────────────────────────────────────────────────┐        │
│ │  #  │                       File                       │                         Change                          │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 1   │ utils/Constant.kt                                │ Add Pidge metafield alias/key constants + webhook flag  │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 2   │ sharedprefsection/MagePrefs.kt                   │ Add Pidge token + expiry save/get methods               │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 3   │ utils/Urls.kt                                    │ Add Pidge base URL + endpoint constants                 │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 4   │ utils/ApiCallInterface.kt                        │ Add Pidge login, order status, rider tracking endpoints │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 5   │ repositories/Repository.kt                       │ Add thin wrapper methods for Pidge calls                │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 6   │ shopifyqueries/Query.kt                          │ Add Pidge order ID metafield to order queries           │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 7   │ ordersection/viewmodels/OrderDetailsViewModel.kt │ Add Pidge LiveData + auth + tracking methods            │        │
│ ├─────┼──────────────────────────────────────────────────┼─────────────────────────────────────────────────────────┤        │
│ │ 8   │ ordersection/activities/OrderStatusPage.kt       │ Replace observers, response handling, add mode switch   │        │
│ └─────┴──────────────────────────────────────────────────┴─────────────────────────────────────────────────────────┘        │
│                                                                                                                             │
│ No new files needed.                                                                                                        │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 1 — utils/Constant.kt                                                                                                  │
│                                                                                                                             │
│ Add after existing alias constants (after line 137 DELIVERY_FEEDBACK_KEY):                                                  │
│                                                                                                                             │
│ // Pidge                                                                                                                    │
│ const val PIDGE_ORDER_ID_KEY = "pidge_order_id"                                                                             │
│ const val PIDGE_ORDER_ID_ALIAS = "pidge_order_id_alias"                                                                     │
│ const val PIDGE_USE_WEBHOOK = false  // false = polling, true = webhook-driven                                              │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 2 — sharedprefsection/MagePrefs.kt                                                                                     │
│                                                                                                                             │
│ Add two new key/value pairs for Pidge auth token and its expiry timestamp:                                                  │
│                                                                                                                             │
│ fun savePidgeToken(token: String) = prefs.edit().putString("PIDGE_TOKEN", token).apply()                                    │
│ fun getPidgeToken(): String? = prefs.getString("PIDGE_TOKEN", null)                                                         │
│ fun savePidgeTokenExpiry(expiryMs: Long) = prefs.edit().putLong("PIDGE_TOKEN_EXPIRY", expiryMs).apply()                     │
│ fun getPidgeTokenExpiry(): Long = prefs.getLong("PIDGE_TOKEN_EXPIRY", 0L)                                                   │
│ fun clearPidgeToken() = prefs.edit().remove("PIDGE_TOKEN").remove("PIDGE_TOKEN_EXPIRY").apply()                             │
│                                                                                                                             │
│ fun isPidgeTokenValid(): Boolean {                                                                                          │
│     val token = getPidgeToken()                                                                                             │
│     val expiry = getPidgeTokenExpiry()                                                                                      │
│     return !token.isNullOrEmpty() && System.currentTimeMillis() < expiry                                                    │
│ }                                                                                                                           │
│                                                                                                                             │
│ Token TTL: store System.currentTimeMillis() + 23 * 60 * 60 * 1000 (23 hours) on save.                                       │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 3 — utils/Urls.kt                                                                                                      │
│                                                                                                                             │
│ Add inside the Data object (alongside SCROLL_ENGINE_BASE_URL block):                                                        │
│                                                                                                                             │
│ // Pidge                                                                                                                    │
│ private const val PIDGE_BASE_URL = "https://api.pidge.in"   // confirm exact base URL with Pidge                            │
│ const val PidgeLogin: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/login"                                            │
│ const val PidgeGetOrder: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}"                                    │
│ const val PidgeGetRiderTracking: String = "$PIDGE_BASE_URL/v1.0/store/channel/vendor/order/{id}/fulfillment/tracking"       │
│                                                                                                                             │
│ // Pidge credentials (move to BuildConfig or secure storage before production)                                              │
│ const val PIDGE_USERNAME = ""   // fill in                                                                                  │
│ const val PIDGE_PASSWORD = ""   // fill in                                                                                  │
│                                                                                                                             │
│ Note: Confirm the exact Pidge base URL before filling in. The Postman doc didn't expose it explicitly.                      │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 4 — utils/ApiCallInterface.kt                                                                                          │
│                                                                                                                             │
│ Add after the /*SCROLL ENGINE START*/ block:                                                                                │
│                                                                                                                             │
│ /*PIDGE START*/                                                                                                             │
│                                                                                                                             │
│ @POST(Urls.PidgeLogin)                                                                                                      │
│ fun pidgeLogin(@Body params: JsonObject): Single<JsonElement>                                                               │
│                                                                                                                             │
│ @GET(Urls.PidgeGetOrder)                                                                                                    │
│ fun pidgeGetOrder(                                                                                                          │
│     @Header("Authorization") token: String,                                                                                 │
│     @Path("id") orderId: String                                                                                             │
│ ): Single<JsonElement>                                                                                                      │
│                                                                                                                             │
│ @GET(Urls.PidgeGetRiderTracking)                                                                                            │
│ fun pidgeGetRiderTracking(                                                                                                  │
│     @Header("Authorization") token: String,                                                                                 │
│     @Path("id") orderId: String                                                                                             │
│ ): Single<JsonElement>                                                                                                      │
│                                                                                                                             │
│ /*PIDGE END*/                                                                                                               │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 5 — repositories/Repository.kt                                                                                         │
│                                                                                                                             │
│ Add thin wrappers after trackShipment():                                                                                    │
│                                                                                                                             │
│ fun pidgeLogin(params: JsonObject): Single<JsonElement> =                                                                   │
│     apiCallInterface.pidgeLogin(params)                                                                                     │
│                                                                                                                             │
│ fun pidgeGetOrder(token: String, orderId: String): Single<JsonElement> =                                                    │
│     apiCallInterface.pidgeGetOrder(token, orderId)                                                                          │
│                                                                                                                             │
│ fun pidgeGetRiderTracking(token: String, orderId: String): Single<JsonElement> =                                            │
│     apiCallInterface.pidgeGetRiderTracking(token, orderId)                                                                  │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 6 — shopifyqueries/Query.kt                                                                                            │
│                                                                                                                             │
│ In the order query (around line 1809 where INVOICE_LINK_ALIAS is defined), add the Pidge metafield:                         │
│                                                                                                                             │
│ .withAlias(Constant.PIDGE_ORDER_ID_ALIAS)                                                                                   │
│ .metafield(Constant.PIDGE_ORDER_ID_KEY,                                                                                     │
│     { s -> s.namespace(Constant.CUSTOM_META_FIELD_NAME_SPACE) }) { m ->                                                     │
│     m.key().value().namespace()                                                                                             │
│ }                                                                                                                           │
│                                                                                                                             │
│ This must be added in every order query that feeds OrderStatusPage — search for INVOICE_LINK_ALIAS in Query.kt and add it   │
│ right after in the same query builder chain.                                                                                │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 7 — ordersection/viewmodels/OrderDetailsViewModel.kt                                                                   │
│                                                                                                                             │
│ 7a. New LiveData fields (add near top with other MutableLiveData):                                                          │
│                                                                                                                             │
│ val pidgeOrderTracking: MutableLiveData<ApiResponse> = MutableLiveData()                                                   │
│ val pidgeShipmentTracking: MutableLiveData<ApiResponse> = MutableLiveData()                                                 │
│ private val _pidgeLoginResponse: MutableLiveData<ApiResponse> = MutableLiveData()                                           │
│ val pidgeLoginResponse: LiveData<ApiResponse> = _pidgeLoginResponse                                                         │
│                                                                                                                             │
│ 7b. Pidge login method:                                                                                                     │
│                                                                                                                             │
│ fun pidgeLogin() {                                                                                                          │
│     val params = JsonObject().apply {                                                                                       │
│         addProperty("username", Urls.PIDGE_USERNAME)                                                                        │
│         addProperty("password", Urls.PIDGE_PASSWORD)                                                                        │
│     }                                                                                                                       │
│     doRetrofitCall(                                                                                                         │
│         repository.pidgeLogin(params),                                                                                      │
│         disposables,                                                                                                        │
│         customResponse = object : CustomResponse {                                                                          │
│             override fun onSuccessRetrofit(result: JsonElement) {                                                           │
│                 try {                                                                                                       │
│                     val token = result.asJsonObject                                                                         │
│                         .getAsJsonObject("data")                                                                            │
│                         .get("token").asString  // "Bearer eyJ..."                                                          │
│                     MagePrefs.savePidgeToken(token)                                                                         │
│                     MagePrefs.savePidgeTokenExpiry(                                                                         │
│                         System.currentTimeMillis() + 23 * 60 * 60 * 1000L                                                   │
│                     )                                                                                                       │
│                     _pidgeLoginResponse.value = ApiResponse.success(result)                                                 │
│                 } catch (ex: Exception) {                                                                                   │
│                     _pidgeLoginResponse.value = ApiResponse.error(ex)                                                       │
│                 }                                                                                                           │
│             }                                                                                                               │
│             override fun onErrorRetrofit(error: Throwable) {                                                                │
│                 _pidgeLoginResponse.value = ApiResponse.error(error)                                                        │
│             }                                                                                                               │
│         },                                                                                                                  │
│         context = context                                                                                                   │
│     )                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ 7c. Pidge order status (replaces trackOrder):                                                                               │
│                                                                                                                             │
│ fun fetchPidgeOrder(pidgeOrderId: String) {                                                                                 │
│     val token = MagePrefs.getPidgeToken() ?: return                                                                         │
│     doRetrofitCall(                                                                                                         │
│         repository.pidgeGetOrder(token, pidgeOrderId),                                                                      │
│         disposables,                                                                                                        │
│         customResponse = object : CustomResponse {                                                                          │
│             override fun onSuccessRetrofit(result: JsonElement) {                                                           │
│                 pidgeOrderTracking.value = ApiResponse.success(result)                                                      │
│             }                                                                                                               │
│             override fun onErrorRetrofit(error: Throwable) {                                                                │
│                 pidgeOrderTracking.value = ApiResponse.error(error)                                                         │
│             }                                                                                                               │
│         },                                                                                                                  │
│         context = context                                                                                                   │
│     )                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ 7d. Pidge rider tracking (replaces trackShipment):                                                                          │
│                                                                                                                             │
│ fun fetchPidgeRiderTracking(pidgeOrderId: String) {                                                                         │
│     val token = MagePrefs.getPidgeToken() ?: return                                                                         │
│     doRetrofitCall(                                                                                                         │
│         repository.pidgeGetRiderTracking(token, pidgeOrderId),                                                              │
│         disposables,                                                                                                        │
│         customResponse = object : CustomResponse {                                                                          │
│             override fun onSuccessRetrofit(result: JsonElement) {                                                           │
│                 pidgeShipmentTracking.value = ApiResponse.success(result)                                                   │
│             }                                                                                                               │
│             override fun onErrorRetrofit(error: Throwable) {                                                                │
│                 pidgeShipmentTracking.value = ApiResponse.error(error)                                                      │
│             }                                                                                                               │
│         },                                                                                                                  │
│         context = context                                                                                                   │
│     )                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Step 8 — ordersection/activities/OrderStatusPage.kt                                                                         │
│                                                                                                                             │
│ This is the largest change. All changes are additive alongside existing Scroll Engine code — the switch is controlled by    │
│ reading the Pidge metafield.                                                                                                │
│                                                                                                                             │
│ 8a. New fields (add near top with existing fields):                                                                         │
│                                                                                                                             │
│ private var pidgeOrderId: String? = null                                                                                    │
│ private var pidgeBackgroundApiCall: Job? = null                                                                             │
│                                                                                                                             │
│ 8b. Pidge order ID extraction (in onCreate, after orderEdge is set):                                                        │
│                                                                                                                             │
│ pidgeOrderId = orderEdge                                                                                                    │
│     ?.withAlias(Constant.PIDGE_ORDER_ID_ALIAS)                                                                              │
│     ?.metafield?.value                                                                                                      │
│     ?.takeIf { it.isNotEmpty() }                                                                                            │
│                                                                                                                             │
│ 8c. Tracking entry point (replaces model?.trackOrder(jsonobject) calls):                                                    │
│                                                                                                                             │
│ Add a helper that decides which provider to use:                                                                            │
│ private fun startTracking() {                                                                                               │
│     val pidgeId = pidgeOrderId                                                                                              │
│     if (!pidgeId.isNullOrEmpty()) {                                                                                         │
│         // Pidge flow                                                                                                       │
│         ensurePidgeTokenThenTrack(pidgeId)                                                                                  │
│     } else {                                                                                                                │
│         // Scroll Engine fallback                                                                                           │
│         model?.trackOrder(jsonobject)                                                                                       │
│     }                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ private fun ensurePidgeTokenThenTrack(pidgeId: String) {                                                                    │
│     if (MagePrefs.isPidgeTokenValid()) {                                                                                    │
│         model?.fetchPidgeOrder(pidgeId)                                                                                     │
│     } else {                                                                                                                │
│         model?.pidgeLogin()                                                                                                 │
│         // pidgeLoginResponse observer calls fetchPidgeOrder on success                                                     │
│     }                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8d. New LiveData observers (add in onCreate alongside existing observers):                                                  │
│                                                                                                                             │
│ model?.pidgeLoginResponse?.observe(this) { response ->                                                                      │
│     when (response.status) {                                                                                                │
│         Status.SUCCESS -> {                                                                                                 │
│             pidgeOrderId?.let { model?.fetchPidgeOrder(it) }                                                                │
│         }                                                                                                                   │
│         else -> {                                                                                                           │
│             // Token fetch failed — fall back to Scroll Engine                                                              │
│             model?.trackOrder(jsonobject)                                                                                   │
│         }                                                                                                                   │
│     }                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ model?.pidgeOrderTracking?.observe(this) { showPidgeOrderStatus(it) }                                                       │
│ model?.pidgeShipmentTracking?.observe(this) { showPidgeRiderLocation(it) }                                                  │
│                                                                                                                             │
│ 8e. showPidgeOrderStatus() — replaces showMap() for Pidge orders:                                                           │
│                                                                                                                             │
│ Maps Pidge GET /order/:id response to existing UI. Response shape:                                                          │
│ { id, status, fulfillment: { status, pickup: { location }, drop: { location }, rider, drop.eta, logs[] } }                  │
│                                                                                                                             │
│ private fun showPidgeOrderStatus(response: ApiResponse?) {                                                                  │
│     try {                                                                                                                   │
│         if (response?.data == null) { hideLoader(); return }                                                                │
│         val data = JSONObject(response.data.toString())                                                                     │
│         val fulfillment = data.optJSONObject("fulfillment") ?: run { hideLoader(); return }                                 │
│         val shipStatus = fulfillment.optString("fulfillment_status", fulfillment.optString("status", ""))                   │
│                                                                                                                             │
│         // Status header label                                                                                              │
│         binding.deliveredtxt.text = getPidgeStatusLabel(shipStatus)                                                         │
│         binding.statussection.setBackgroundResource(getPidgeStatusBackground(shipStatus))                                   │
│                                                                                                                             │
│         // Locations                                                                                                        │
│         val pickup = fulfillment.optJSONObject("pickup")                                                                    │
│         val drop = fulfillment.optJSONObject("drop")                                                                        │
│         storelat = pickup?.optJSONObject("location")?.optString("latitude", "") ?: ""                                       │
│         storelong = pickup?.optJSONObject("location")?.optString("longitude", "") ?: ""                                     │
│         userlat = drop?.optJSONObject("location")?.optString("latitude", "") ?: ""                                          │
│         userlong = drop?.optJSONObject("location")?.optString("longitude", "") ?: ""                                        │
│                                                                                                                             │
│         // Rider                                                                                                            │
│         val rider = fulfillment.optJSONObject("rider")                                                                      │
│         if (rider != null) {                                                                                                │
│             binding.deliveryName.text = rider.optString("name")                                                             │
│             binding.assignmancallsect.setOnClickListener {                                                                  │
│                 startActivity(Intent(Intent.ACTION_DIAL).apply {                                                            │
│                     data = Uri.parse("tel:${rider.optString("mobile")}")                                                    │
│                 })                                                                                                          │
│             }                                                                                                               │
│         }                                                                                                                   │
│                                                                                                                             │
│         // Delivery timestamp from logs (DELIVERED entry)                                                                   │
│         if (shipStatus == "DELIVERED" || shipStatus == "PARTIAL_DELIVERED") {                                               │
│             manageTimerLabelForDeliveredOrderFromPidge(fulfillment)                                                         │
│         }                                                                                                                   │
│                                                                                                                             │
│         // Shipment ID (use Pidge order id as shipment reference for feedback)                                              │
│         shipmentId = pidgeOrderId                                                                                           │
│         getFeedBack("showPidgeOrderStatus")                                                                                 │
│                                                                                                                             │
│         // Trigger rider location fetch based on mode                                                                       │
│         if (shipStatus != "DELIVERED" && shipStatus != "CANCELLED") {                                                       │
│             if (!Constant.PIDGE_USE_WEBHOOK) {                                                                              │
│                 startPidgePolling()                                                                                         │
│             }                                                                                                               │
│             // If webhook mode: wait for FCM push — no polling                                                              │
│         }                                                                                                                   │
│                                                                                                                             │
│         hideLoader()                                                                                                        │
│         hideTabSwitchLoader()                                                                                               │
│     } catch (ex: Exception) {                                                                                               │
│         exceptionLogFirebase(ex.printStackTrace().toString(), TAG, MagePrefs.getCustomerID())                               │
│         hideLoader()                                                                                                        │
│     }                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8f. showPidgeRiderLocation() — replaces showDriverLocation() for Pidge:                                                     │
│                                                                                                                             │
│ Maps GET /order/:id/fulfillment/tracking response:                                                                          │
│ { data: { rider: { name, mobile }, status, location: { latitude, longitude } } }                                            │
│                                                                                                                             │
│ private fun showPidgeRiderLocation(response: ApiResponse?) {                                                                │
│     try {                                                                                                                   │
│         if (response?.data == null) { stopRefreshAnimation(); return }                                                      │
│         val repo = JSONObject(response.data.toString())                                                                     │
│         if (!repo.getBoolean("status")) { stopRefreshAnimation(); return }                                                  │
│         val data = repo.getJSONObject("data")                                                                               │
│         val shipStatus = data.optString("status", "")                                                                       │
│         val rider = data.optJSONObject("rider")                                                                             │
│         val location = data.optJSONObject("location")                                                                       │
│                                                                                                                             │
│         shipmentStatus = shipStatus                                                                                         │
│                                                                                                                             │
│         // Update driver name and call button                                                                               │
│         if (rider != null && rider.length() > 0) {                                                                          │
│             binding.deliveryName.text = rider.optString("name")                                                             │
│             binding.assignmancallsect.visibility = View.VISIBLE                                                             │
│             binding.assignmancallsect.setOnClickListener {                                                                  │
│                 startActivity(Intent(Intent.ACTION_DIAL).apply {                                                            │
│                     data = Uri.parse("tel:${rider.optString("mobile")}")                                                    │
│                 })                                                                                                          │
│             }                                                                                                               │
│         }                                                                                                                   │
│                                                                                                                             │
│         // Map markers + polyline                                                                                           │
│         if (location != null && isMapInitialized) {                                                                         │
│             val driverLat = location.optDouble("latitude")                                                                  │
│             val driverLng = location.optDouble("longitude")                                                                 │
│             val driverLatLng = LatLng(driverLat, driverLng)                                                                 │
│             val userLatLng = LatLng(userlat.toDoubleOrNull() ?: 0.0, userlong.toDoubleOrNull() ?: 0.0)                      │
│             val storeLatLng = LatLng(storelat.toDoubleOrNull() ?: 0.0, storelong.toDoubleOrNull() ?: 0.0)                   │
│                                                                                                                             │
│             mMap.clear()                                                                                                    │
│             if (shipStatus == "OUT_FOR_DELIVERY") {                                                                         │
│                 val icon = BitmapDescriptorFactory.fromBitmap(                                                              │
│                     ResourcesCompat.getDrawable(resources, R.drawable.deliverytruck, null)?.toBitmap()!!                    │
│                 )                                                                                                           │
│                                                                                                                             │
│ mMap.addMarker(MarkerOptions().position(userLatLng).title(getString(R.string.delivery_address)).icon(userBitMap))           │
│                 mMap.addMarker(MarkerOptions().position(driverLatLng).title(rider?.optString("name") ?:                     │
│ "Driver").icon(icon))                                                                                                       │
│                 binding.reload.visible()                                                                                    │
│                 getDocument(start = driverLatLng, middle = storeLatLng, end = userLatLng, "driving")                        │
│             } else {                                                                                                        │
│                 mMap.addMarker(MarkerOptions().position(storeLatLng).title("Store").icon(storeBitMap))                      │
│                                                                                                                             │
│ mMap.addMarker(MarkerOptions().position(userLatLng).title(getString(R.string.delivery_address)).icon(userBitMap))           │
│                 val curvedPoints = getCurvePoints(storeLatLng, userLatLng)                                                  │
│                                                                                                                             │
│ mMap.addPolyline(PolylineOptions().addAll(curvedPoints).width(5f).color(Color.BLACK).pattern(listOf(Dash(20f), Gap(10f))))  │
│                 binding.reload.gone()                                                                                       │
│             }                                                                                                               │
│             val bounds = LatLngBounds.Builder().include(storeLatLng).include(userLatLng).build()                            │
│             mMap.animateCamera(CameraUpdateFactory.newLatLngBounds(bounds, 100))                                            │
│         }                                                                                                                   │
│                                                                                                                             │
│         // Schedule next poll (polling mode only)                                                                           │
│         if (!Constant.PIDGE_USE_WEBHOOK && shipStatus != "DELIVERED" && shipStatus != "CANCELLED") {                        │
│             pidgeBackgroundApiCall = lifecycleScope.launch(Dispatchers.IO) {                                                │
│                 delay(8000)                                                                                                 │
│                 pidgeOrderId?.let { model?.fetchPidgeRiderTracking(it) }                                                    │
│             }                                                                                                               │
│         }                                                                                                                   │
│                                                                                                                             │
│         stopRefreshAnimation()                                                                                              │
│         hideTabSwitchLoader()                                                                                               │
│         hideLoader()                                                                                                        │
│     } catch (ex: Exception) {                                                                                               │
│         exceptionLogFirebase(ex.printStackTrace().toString(), TAG, MagePrefs.getCustomerID())                               │
│         stopRefreshAnimation()                                                                                              │
│     }                                                                                                                       │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8g. Polling starter:                                                                                                        │
│                                                                                                                             │
│ private fun startPidgePolling() {                                                                                           │
│     pidgeBackgroundApiCall?.cancel()                                                                                        │
│     pidgeOrderId?.let { model?.fetchPidgeRiderTracking(it) }                                                                │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8h. Webhook mode — FCM handler:                                                                                             │
│                                                                                                                             │
│ When PIDGE_USE_WEBHOOK = true, the FCM push from our backend should trigger a re-fetch.                                     │
│ In onNewIntent() or onResume(), check for a "pidge_update" flag in the intent and call startTracking().                     │
│ (Backend webhook receiver → FCM → app re-fetches from Pidge directly.)                                                      │
│                                                                                                                             │
│ 8i. Status label + colour helpers (add as private functions):                                                               │
│                                                                                                                             │
│ private fun getPidgeStatusLabel(status: String): String = when (status) {                                                   │
│     "CREATED"          -> "Order Confirmed"                                                                                 │
│     "OUT_FOR_PICKUP"   -> "Rider Heading to Store"                                                                          │
│     "PICKED_UP"        -> "Order Picked Up"                                                                                 │
│     "OUT_FOR_DELIVERY" -> "Out for Delivery"                                                                                │
│     "DELIVERED"        -> "Order Delivered"                                                                                 │
│     "CANCELLED"        -> "Order Cancelled"                                                                                 │
│     else               -> "Tracking your order"                                                                             │
│ }                                                                                                                           │
│                                                                                                                             │
│ private fun getPidgeStatusBackground(status: String): Int = when (status) {                                                 │
│     "DELIVERED"        -> R.drawable.gradiantgreen  // confirm drawable name                                                │
│     "CANCELLED"        -> R.drawable.gradiantred                                                                            │
│     else               -> R.drawable.gradiantyellow // yellow for in-progress                                               │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8j. Delivery timestamp from Pidge logs:                                                                                     │
│                                                                                                                             │
│ private fun manageTimerLabelForDeliveredOrderFromPidge(fulfillment: JSONObject) {                                           │
│     try {                                                                                                                   │
│         isDeliveredMarkedFromScrollEngine = true                                                                            │
│         supportFragmentManager.beginTransaction().hide(mapFragment).commit()                                                │
│         binding.statusicn.visibility = View.VISIBLE                                                                         │
│         binding.deliveredtxt.text = "Order Delivered"                                                                       │
│         binding.assignmessagesect.gone()                                                                                    │
│         binding.assignmsgsect.gone()                                                                                        │
│         binding.orderdiv.gone()                                                                                             │
│         binding.assignmancallsect.gone()                                                                                    │
│         binding.addFeedback.root.visible()                                                                                  │
│         binding.reload.gone()                                                                                               │
│                                                                                                                             │
│         // Extract delivery time from drop.timestamp                                                                        │
│         val dropTimestamp = fulfillment.optJSONObject("drop")?.optString("timestamp", "")                                   │
│         if (!dropTimestamp.isNullOrEmpty()) {                                                                               │
│             binding.delivedtime.text = "Your order was delivered ${formatTimeStamp(dropTimestamp)}"                         │
│             binding.delivedtime.visibility = View.VISIBLE                                                                   │
│             binding.delivedtimeSection.visibility = View.VISIBLE                                                           │
│         }                                                                                                                   │
│     } catch (_: Exception) {}                                                                                               │
│ }                                                                                                                           │
│                                                                                                                             │
│ 8k. Replace all model?.trackOrder(jsonobject) call sites with startTracking():                                              │
│                                                                                                                             │
│ Search for model?.trackOrder(jsonobject) in OrderStatusPage.kt — replace each with startTracking().                         │
│                                                                                                                             │
│ 8l. Cancel Pidge polling on destroy (add to onDestroy()):                                                                   │
│                                                                                                                             │
│ pidgeBackgroundApiCall?.cancel()                                                                                            │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Status Mapping Reference                                                                                                    │
│                                                                                                                             │
│ ┌──────────────────┬──────────────────────────┬───────────────┬────────────────────────────────────────┬─────────┐          │
│ │   Pidge Status   │       Header Text        │ Header Colour │                  Map                   │ Polling │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ CREATED          │ "Order Confirmed"        │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ OUT_FOR_PICKUP   │ "Rider Heading to Store" │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ PICKED_UP        │ "Order Picked Up"        │ Yellow        │ Store + delivery markers + Bezier      │ Yes     │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ OUT_FOR_DELIVERY │ "Out for Delivery"       │ Yellow        │ Truck marker + Google Directions route │ Yes     │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ DELIVERED        │ "Order Delivered"        │ Green         │ Hidden                                 │ No      │          │
│ ├──────────────────┼──────────────────────────┼───────────────┼────────────────────────────────────────┼─────────┤          │
│ │ CANCELLED        │ "Order Cancelled"        │ Red           │ Cleared                                │ No      │          │
│ └──────────────────┴──────────────────────────┴───────────────┴────────────────────────────────────────┴─────────┘          │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Scroll Engine Fallback                                                                                                      │
│                                                                                                                             │
│ If pidgeOrderId is null/empty (order not yet on Pidge, or metafield not set):                                               │
│ - startTracking() falls through to model?.trackOrder(jsonobject) → existing Scroll Engine flow unchanged                    │
│ - No existing Scroll Engine code is removed — full backward compatibility                                                   │
│                                                                                                                             │
│ ---                                                                                                                         │
│ What Remains Pending (confirm with Pidge)                                                                                   │
│                                                                                                                             │
│ - Exact Pidge base URL                                                                                                      │
│ - Exact metafield key name they will use to store Pidge order ID on Shopify                                                 │
│ - Driver feedback API (submit + get rating)                                                                                 │
│ - PARTIAL_DELIVERED and CANCELLED exact status strings                                                                      │
│ - track_code — full URL or code only                                                                                        │
│ - Multi-shipment support                                                                                                    │
│                                                                                                                             │
│ ---                                                                                                                         │
│ Verification Steps                                                                                                          │
│                                                                                                                             │
│ 1. Build assembleDebug — no compile errors                                                                                  │
│ 2. Open an order that has pidge_order_id metafield set → confirm Pidge flow is entered                                      │
│ 3. Open an order without the metafield → confirm Scroll Engine flow is unchanged                                            │
│ 4. Toggle Constant.PIDGE_USE_WEBHOOK = true → confirm polling loop does not start                                           │
│ 5. Toggle Constant.PIDGE_USE_WEBHOOK = false → confirm 8s polling starts after initial load                                 │
│ 6. Simulate expired token (clear MagePrefs) → confirm re-login runs before tracking                                         │
│ 7. Confirm map markers render correctly for each status                                                                     │
│ 8. Confirm manageTimerLabelForDeliveredOrderFromPidge shows correct delivery time                                           │
╰─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
