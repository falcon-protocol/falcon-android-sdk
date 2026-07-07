# Falcon Android SDK

The Falcon Android SDK lets you embed inline ad placements inside your Android app. Placements are rendered in a self-sizing WebView-backed view that drives its own height to match the rendered content and fires lifecycle callbacks to your app. The SDK mirrors the Falcon iOS SDK surface, so teams integrating both platforms find the same API shape.

Full integration guide: [docs.falconlabs.com/integration-guide/android](https://docs.falconlabs.com/integration-guide/android)

---

## Requirements

- **minSdk 24** (Android 7.0) or higher.
- The library manifest declares `android.permission.INTERNET`; the manifest merger adds it to your app automatically — nothing to declare yourself.
- Call all `Falcon` methods on the **main thread**; all callbacks are delivered on the main thread.

---

## Installation

The SDK is published to **Maven Central** (`us.falconlabs:falcon`) — no extra repository block is
needed:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}

// app/build.gradle.kts
dependencies {
    implementation("us.falconlabs:falcon:1.1.0")
}
```

> **Note for existing integrations:** versions up to 1.0.0 were served from this repository's
> raw-URL Maven repo (`https://raw.githubusercontent.com/falcon-protocol/falcon-android-sdk/main/repo`).
> That repo is still maintained as a byte-identical mirror of Central, so existing consumers keep
> working — but new integrations should use Maven Central alone.

The SDK's runtime dependencies (`androidx.browser` for Chrome Custom Tabs, Compose UI/foundation via the Compose BOM) are carried by the POM and resolved transitively — you do not need to declare them yourself.

---

## Initialize the SDK

Call `Falcon.initSdk` once at application launch, before any placement is shown. The recommended place is `Application.onCreate`.

```kotlin
import android.app.Application
import us.falconlabs.falcon.Falcon

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        Falcon.initSdk(this, "YOUR_API_KEY")
    }
}
```

An optional `FalconConfig` parameter controls the backend base URL and placement mapping:

```kotlin
import us.falconlabs.falcon.Falcon
import us.falconlabs.falcon.FalconConfig

Falcon.initSdk(
    context = this,
    apiKey  = "YOUR_API_KEY",
    config  = FalconConfig(
        baseUrl          = "https://pr.falconlabs.us",
        placementMapping = mapOf(
            "APP_NATIVE_ESSENTIAL_0.1/ORDER_STATUS" to "your-placement-id",
        ),
    ),
)
```

`placementMapping` resolves the Falcon placement ID from the identity supplied in `execute`. Lookup order: `"<layout_id>/<view>"`, then `"<view>"`, then `"<layout_id>"`. When no entry matches, the `view` string is sent as the placement ID.

The `apiKey` is the publisher-scoped token issued by Falcon, designed for client-side use; it is not a secret API key.

---

## Add an Inline Placement

### XML layout

Add `FalconEmbeddedView` to your layout. Set `layout_height="0dp"` — the view drives its own height to match the rendered content:

```xml
<us.falconlabs.falcon.FalconEmbeddedView
    android:id="@+id/falcon_view"
    android:layout_width="match_parent"
    android:layout_height="0dp" />
```

In a vertical `LinearLayout`, `layout_height="0dp"` lets the view size itself to content. In a `ConstraintLayout`, `0dp` means match-constraint: set `app:layout_constraintTop_toTopOf`, `app:layout_constraintStart_toStartOf`, and `app:layout_constraintEnd_toEndOf` and leave `layout_height="0dp"` so the SDK can drive the height freely.

### Programmatic alternative

```kotlin
import us.falconlabs.falcon.FalconEmbeddedView
import android.widget.LinearLayout

val falconView = FalconEmbeddedView(context)
val lp = LinearLayout.LayoutParams(
    LinearLayout.LayoutParams.MATCH_PARENT,
    0, // height driven by the SDK
)
container.addView(falconView, lp)
```

---

## Execute

Call `Falcon.execute` to load and show a placement. Pass user and placement details in the `attributes` map and a reference to the view in `placement`:

```kotlin
import us.falconlabs.falcon.Falcon
import us.falconlabs.falcon.FalconPlacement

val attributes = mapOf(
    "user_details" to mapOf(
        "email"      to "user@example.com",
        "first_name" to "First",
        "last_name"  to "Last",
    ),
    "placement_details" to mapOf(
        "layout_id" to "APP_NATIVE_ESSENTIAL_0.1",
        "view"      to "ORDER_STATUS",
    ),
)

Falcon.execute(
    attributes = attributes,
    placement  = FalconPlacement.Inline(falconView),
)
```

All parameters other than `attributes` and `placement` are optional. The full signature with every optional parameter:

```kotlin
Falcon.execute(
    attributes                    = attributes,
    placement                     = FalconPlacement.Inline(falconView),
    style                         = null,
    onLoad                        = { /* placement visible */ },
    onUnload                      = { /* placement removed */ },
    onError                       = { error -> /* handle error */ },
    onShouldShowLoadingIndicator  = { loadingSpinner.visibility = View.VISIBLE },
    onShouldHideLoadingIndicator  = { loadingSpinner.visibility = View.GONE },
    isSandbox                     = false,
)
```

---

## Sandbox Mode

Set `isSandbox = true` during development so Falcon can differentiate test sessions from production traffic. Set it to `false` before shipping.

```kotlin
Falcon.execute(
    attributes = attributes,
    placement  = FalconPlacement.Inline(falconView),
    isSandbox  = true,
)
```

---

## Style Param

Use the `style` parameter to customise placement colours and typography. All fields are optional; omitting a field leaves the placement default in place.

```kotlin
import android.graphics.Color
import us.falconlabs.falcon.FalconStyle

Falcon.execute(
    attributes = attributes,
    placement  = FalconPlacement.Inline(falconView),
    style = FalconStyle(
        widgetBackgroundColor       = 0x00000000.toInt(), // transparent
        slotBackgroundColor         = 0xFFFFFFFF.toInt(), // white
        slotPadding                 = 8,
        acceptButtonBackgroundColor = Color.parseColor("#008363"),
        acceptButtonTextColor       = 0xFFFFFFFF.toInt(), // white
        promoCodeBackgroundColor    = Color.parseColor("#2DA784"),
        fontFamily                  = "Roboto",
    ),
)
```

| Field | Type | Description | Default |
|---|---|---|---|
| `widgetBackgroundColor` | ARGB `@ColorInt` | Background colour of the widget | `null` (placement default) |
| `slotBackgroundColor` | ARGB `@ColorInt` | Background colour of each slot | `null` (placement default) |
| `slotPadding` | `Int` | Forwarded verbatim as CSS pixels to the web render layer | `null` (placement default) |
| `acceptButtonBackgroundColor` | ARGB `@ColorInt` | Background colour of the accept button | `null` (placement default) |
| `acceptButtonTextColor` | ARGB `@ColorInt` | Text colour of the accept button | `null` (placement default) |
| `promoCodeBackgroundColor` | ARGB `@ColorInt` | Background colour of the promo code transition interface | `null` (placement default) |
| `fontFamily` | `String` | Font family name forwarded to the web render layer (e.g. `"Roboto"`) | `null` (placement default) |

Colour values are standard Android ARGB `@ColorInt` integers (as returned by `Color.parseColor`, `ContextCompat.getColor`, `"#RRGGBB".toColorInt()`, etc.).

---

## Callbacks

All callbacks fire on the main thread and are safe to update the UI from.

### `onLoad`
Called when the placement has loaded and is visible. Passes no arguments, returns no value.

### `onUnload`
Called when the placement has been completed and removed from the UI. Passes no arguments, returns no value.

### `onError`
Called when an error occurs while loading the placement. `onLoad` and `onUnload` will not be called when an error occurs. Receives an `Exception` instance.

**Error types:**

| Error | Meaning |
|---|---|
| `FalconError.InitNotCalled` | `Falcon.execute` was called before `Falcon.initSdk` |
| `FalconError.PlacementLoadError` | The placement failed to load from the Falcon backend |

### `onShouldShowLoadingIndicator`
Called when the placement has begun loading. Use this to display a progress indicator.

### `onShouldHideLoadingIndicator`
Called when the placement has either loaded or failed to load. Use this to hide any progress indicator.

**No-fill behaviour:** when there are no offers to show, the view stays collapsed at height 0, `onShouldHideLoadingIndicator` is called, and neither `onLoad` nor `onError` fires.

---

## Events

Subscribe to per-layout lifecycle events via `Falcon.events`. Pass the same `layout_id` used in the `attributes` map of `execute`:

```kotlin
import us.falconlabs.falcon.Falcon
import us.falconlabs.falcon.FalconEvent

Falcon.events("APP_NATIVE_ESSENTIAL_0.1") { event ->
    when (event) {
        is FalconEvent.PlacementInteractive -> { /* placement is viewable */ }
        is FalconEvent.PlacementCompleted   -> { /* user engaged; placement removed */ }
        is FalconEvent.PlacementFailure     -> { /* event.error contains details */ }
    }
}
```

**Event types:**

| Event | Meaning |
|---|---|
| `PlacementInteractive` | Over 50% of the placement has been visible in the window for 1 second |
| `PlacementCompleted` | There was one hero slot and the user engaged with it; the placement has been removed |
| `PlacementFailure` | The placement failed to load; inspect `event.error` for details |

**Remove the handler** when your component is destroyed to avoid retaining a reference to the Activity or other context:

```kotlin
override fun onDestroy() {
    Falcon.removeEventsHandler("APP_NATIVE_ESSENTIAL_0.1")
    super.onDestroy()
}
```

Handlers retain whatever they capture (often an Activity) until explicitly removed or overwritten by a new call to `Falcon.events` with the same layout ID.

---

## Jetpack Compose

Use `FalconEmbedded` to embed a placement in a Compose UI. The parameters mirror `Falcon.execute`:

```kotlin
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import us.falconlabs.falcon.FalconEmbedded

@Composable
fun OrderConfirmationScreen() {
    var status by remember { mutableStateOf("loading…") }

    Column(Modifier.fillMaxSize().padding(16.dp)) {
        FalconEmbedded(
            attributes = mapOf(
                "user_details" to mapOf(
                    "email"      to "user@example.com",
                    "first_name" to "First",
                    "last_name"  to "Last",
                ),
                "placement_details" to mapOf(
                    "layout_id" to "APP_NATIVE_ESSENTIAL_0.1",
                    "view"      to "ORDER_STATUS",
                ),
            ),
            onLoad    = { status = "loaded" },
            onUnload  = { status = "unloaded" },
            onError   = { status = "error: ${it.message}" },
            isSandbox = true,
        )
        Text("status: $status")
    }
}
```

The `config` parameter is accepted for signature parity but is inert at runtime — SDK configuration is set via `Falcon.initSdk`.

Attribute changes after first composition are intentionally ignored. To force a fresh placement load when attributes change, wrap in `key(...)` and change the key:

```kotlin
key(userId) {
    FalconEmbedded(attributes = attributesForUser(userId), ...)
}
```

---

## Behavior Notes

- **Link handling:** user taps on placement links open in Chrome Custom Tabs. Other URL schemes are dispatched via `ACTION_VIEW` (deep links). Dangerous schemes (e.g. `javascript:`) are blocked and silently dropped.
- **Navigation policy:** navigation away from the placement's origin inside the WebView is blocked. The placement runs entirely within its own origin.
- **Click gesture gate:** click events from ad content are honored only within a short window after a genuine user touch on the ad view. Remote content cannot silently open browsers or deep links without a real interaction.
- **Reserved JS interface:** the JavaScript interface name `Android` is used by the SDK's bridge inside the placement WebView. Do not register a JavaScript interface with this name on the same WebView.
- **WebView debugging:** the SDK never enables WebView debugging. To inspect placements via `chrome://inspect`, call `WebView.setWebContentsDebuggingEnabled(true)` in your own debug builds (it is an app-global flag, so it is your call to make).
- **R8/ProGuard:** no host configuration needed — the AAR ships a consumer rule that keeps the JS-interface method.

---

## Support

Questions or integration help: contact your Falcon representative.
