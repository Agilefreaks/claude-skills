# Environment + dev tools — canonical file contents

This reference holds the verbatim Kotlin source for the qa-only env-override layer in `core/data` and the shake/broadcast dev-tools UI in `core/ui-mobile` (and `core/ui-tv` for TV builds). The overview, file map, and wiring summary live in `SKILL.md`'s "Environment + dev tools (qa-only)" section.

Substitute `<root-pkg>` with the project's root Kotlin package. Read `SKILL.md` first — that section explains *why* each piece exists and where it gets wired.

## `core/data/.../env/EnvironmentConfig.kt` — interface

```kotlin
package <root-pkg>.core.data.env

import kotlinx.coroutines.flow.StateFlow

interface EnvironmentConfig {
    /** Live base URL — emits `BuildConfig.API_BASE_URL` until the user overrides it. */
    val apiBaseUrl: StateFlow<String>

    /** Pre-canned options the dialog uses for the Staging / Prod radio buttons. */
    val stagingUrl: String
    val prodUrl: String

    suspend fun setApiBaseUrl(url: String)

    /** Drop the override and fall back to `BuildConfig.API_BASE_URL`. */
    suspend fun reset()
}
```

## `core/data/.../env/EnvironmentConfigImpl.kt` — DataStore-backed implementation

```kotlin
package <root-pkg>.core.data.env

import android.content.Context
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import <root-pkg>.core.data.BuildConfig
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.launch

private val Context.envDataStore by preferencesDataStore(name = "env_config")
private val API_BASE_URL_KEY = stringPreferencesKey("api_base_url")

class EnvironmentConfigImpl(
    private val context: Context,
    private val scope: CoroutineScope,
    override val stagingUrl: String,
    override val prodUrl: String,
) : EnvironmentConfig {
    private val _apiBaseUrl = MutableStateFlow(BuildConfig.API_BASE_URL)
    override val apiBaseUrl: StateFlow<String> = _apiBaseUrl.asStateFlow()

    init {
        scope.launch {
            val stored = context.envDataStore.data.map { it[API_BASE_URL_KEY] }.first()
            if (!stored.isNullOrBlank()) _apiBaseUrl.value = stored
        }
    }

    override suspend fun setApiBaseUrl(url: String) {
        _apiBaseUrl.value = url
        context.envDataStore.edit { it[API_BASE_URL_KEY] = url }
    }

    override suspend fun reset() {
        _apiBaseUrl.value = BuildConfig.API_BASE_URL
        context.envDataStore.edit { it.remove(API_BASE_URL_KEY) }
    }
}
```

## `core/data/.../env/di/EnvironmentModule.kt` — Koin module

```kotlin
package <root-pkg>.core.data.env.di

import <root-pkg>.core.data.BuildConfig
import <root-pkg>.core.data.env.EnvironmentConfig
import <root-pkg>.core.data.env.EnvironmentConfigImpl
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.SupervisorJob
import org.koin.dsl.module

val environmentModule = module {
    single<EnvironmentConfig> {
        EnvironmentConfigImpl(
            context = get(),
            scope = CoroutineScope(SupervisorJob()),
            // The compile-time flavor URL is also the canonical "Staging" choice on qa builds.
            // For prod builds, IS_QA is false and the dialog is never shown — these stay unused.
            stagingUrl = "https://api-staging.example.com/",
            prodUrl = "https://api.example.com/",
        )
    }
}
```

The literal Staging/Prod URLs in `environmentModule` must match what the wizard wrote into `gradle.properties` (`<project>.qaApiBaseUrl`, `<project>.prodApiBaseUrl`) — write them once into the user's config and reuse the same values here so the dialog labels match the actual flavor defaults.

## `core/data/.../network/EnvironmentBaseUrlInterceptor.kt`

```kotlin
package <root-pkg>.core.data.network

import <root-pkg>.core.data.env.EnvironmentConfig
import okhttp3.HttpUrl.Companion.toHttpUrlOrNull
import okhttp3.Interceptor
import okhttp3.Response

/**
 * Rewrites every outgoing request's scheme/host/port to match the current EnvironmentConfig URL.
 * The request path/query stays as the caller built it; only the origin is swapped. This lets the
 * Retrofit base URL stay constant (the BuildConfig default) while runtime switches honor the dialog.
 */
class EnvironmentBaseUrlInterceptor(
    private val environmentConfig: EnvironmentConfig,
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val current = environmentConfig.apiBaseUrl.value.toHttpUrlOrNull() ?: return chain.proceed(chain.request())
        val original = chain.request()
        val newUrl = original.url.newBuilder()
            .scheme(current.scheme)
            .host(current.host)
            .port(current.port)
            .build()
        return chain.proceed(original.newBuilder().url(newUrl).build())
    }
}
```

Register it as the **first** interceptor on the OkHttp client so subsequent auth/logging interceptors see the rewritten URL.

## `core/ui-mobile/.../dev/ShakeDetector.kt`

```kotlin
package <root-pkg>.core.ui.mobile.dev

import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.rememberUpdatedState
import androidx.compose.ui.platform.LocalContext
import kotlin.math.sqrt

private const val SHAKE_THRESHOLD_G = 2.7f       // ~2.7g spike, robust to walking
private const val MIN_INTERVAL_MS = 1_000L       // debounce so one shake fires once

@Composable
fun ShakeListener(onShake: () -> Unit) {
    val context = LocalContext.current
    val latestOnShake by rememberUpdatedState(onShake)

    DisposableEffect(context) {
        val sensorManager = context.getSystemService(Context.SENSOR_SERVICE) as? SensorManager
        val accelerometer = sensorManager?.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        if (sensorManager == null || accelerometer == null) {
            return@DisposableEffect onDispose { }
        }
        var lastShakeAt = 0L
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                val gX = event.values[0] / SensorManager.GRAVITY_EARTH
                val gY = event.values[1] / SensorManager.GRAVITY_EARTH
                val gZ = event.values[2] / SensorManager.GRAVITY_EARTH
                val gForce = sqrt(gX * gX + gY * gY + gZ * gZ)
                if (gForce >= SHAKE_THRESHOLD_G) {
                    val now = System.currentTimeMillis()
                    if (now - lastShakeAt >= MIN_INTERVAL_MS) {
                        lastShakeAt = now
                        latestOnShake()
                    }
                }
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) = Unit
        }
        sensorManager.registerListener(listener, accelerometer, SensorManager.SENSOR_DELAY_UI)
        onDispose { sensorManager.unregisterListener(listener) }
    }
}
```

## `core/ui-mobile/.../dev/DevToolsBroadcastListener.kt`

```kotlin
package <root-pkg>.core.ui.mobile.dev

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.rememberUpdatedState
import androidx.compose.ui.platform.LocalContext
import androidx.core.content.ContextCompat

/**
 * Listens for the dev-tools broadcast (`<applicationId>.OPEN_DEV_TOOLS`) and invokes [onTrigger].
 * The action is namespaced with the app's packageName so receivers across apps don't collide.
 */
@Composable
fun DevToolsBroadcastListener(onTrigger: () -> Unit) {
    val context = LocalContext.current
    val latestOnTrigger by rememberUpdatedState(onTrigger)

    DisposableEffect(context) {
        val action = "${context.packageName}.OPEN_DEV_TOOLS"
        val receiver = object : BroadcastReceiver() {
            override fun onReceive(c: Context?, intent: Intent?) {
                if (intent?.action == action) latestOnTrigger()
            }
        }
        ContextCompat.registerReceiver(
            context,
            receiver,
            IntentFilter(action),
            ContextCompat.RECEIVER_NOT_EXPORTED,
        )
        onDispose { context.unregisterReceiver(receiver) }
    }
}
```

## `core/ui-mobile/.../dev/DevToolsHost.kt`

```kotlin
package <root-pkg>.core.ui.mobile.dev

import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue

/**
 * Wraps the app content. When [enabled] is false this is a zero-cost passthrough — prod builds
 * pay nothing. When true, mounts the shake listener + dev-tools broadcast receiver and shows
 * [EnvSelectorDialog] on either trigger.
 */
@Composable
fun DevToolsHost(
    enabled: Boolean,
    content: @Composable () -> Unit,
) {
    if (!enabled) {
        content()
        return
    }
    var showDialog by remember { mutableStateOf(false) }
    ShakeListener(onShake = { showDialog = true })
    DevToolsBroadcastListener(onTrigger = { showDialog = true })
    content()
    if (showDialog) {
        EnvSelectorDialog(onDismiss = { showDialog = false })
    }
}
```

## `core/ui-mobile/.../dev/EnvSelectorDialog.kt`

```kotlin
package <root-pkg>.core.ui.mobile.dev

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.selection.selectable
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.RadioButton
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import <root-pkg>.core.data.env.EnvironmentConfig
import kotlinx.coroutines.launch
import org.koin.compose.koinInject

@Composable
fun EnvSelectorDialog(
    modifier: Modifier = Modifier,
    onDismiss: () -> Unit,
    environmentConfig: EnvironmentConfig = koinInject(),
) {
    val current by environmentConfig.apiBaseUrl.collectAsState()
    var selected by remember { mutableStateOf(current) }
    var customUrl by remember { mutableStateOf(if (selected !in setOf(environmentConfig.stagingUrl, environmentConfig.prodUrl)) selected else "") }
    val scope = rememberCoroutineScope()

    AlertDialog(
        modifier = modifier,
        onDismissRequest = onDismiss,
        title = { Text("Select API environment") },
        text = {
            Column {
                EnvOption("Staging", environmentConfig.stagingUrl, selected) { selected = environmentConfig.stagingUrl }
                EnvOption("Production", environmentConfig.prodUrl, selected) { selected = environmentConfig.prodUrl }
                EnvOption("Custom URL", customUrl, selected) { selected = customUrl }
                OutlinedTextField(
                    value = customUrl,
                    onValueChange = { customUrl = it; if (selected !in setOf(environmentConfig.stagingUrl, environmentConfig.prodUrl)) selected = it },
                    label = { Text("Custom base URL") },
                    modifier = Modifier.padding(top = 8.dp),
                )
            }
        },
        confirmButton = {
            TextButton(onClick = {
                scope.launch { environmentConfig.setApiBaseUrl(selected) }
                onDismiss()
            }) { Text("Apply") }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancel") } },
    )
}

@Composable
private fun EnvOption(label: String, value: String, selected: String, onSelect: () -> Unit) {
    androidx.compose.foundation.layout.Row(
        modifier = Modifier.selectable(selected = selected == value, onClick = onSelect).padding(vertical = 4.dp),
    ) {
        RadioButton(selected = selected == value, onClick = onSelect)
        Text("$label  ($value)", modifier = Modifier.padding(start = 8.dp))
    }
}
```

## TV variant — `core/ui-tv/.../dev/DevToolsHost.kt`

TV remotes don't shake, so the TV `DevToolsHost` skips `ShakeListener` and keeps only the broadcast receiver:

```kotlin
package <root-pkg>.core.ui.tv.dev

import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import <root-pkg>.core.ui.base.dev.DevToolsBroadcastListener
import <root-pkg>.core.ui.base.dev.EnvSelectorDialog

@Composable
fun DevToolsHost(enabled: Boolean, content: @Composable () -> Unit) {
    if (!enabled) { content(); return }
    var showDialog by remember { mutableStateOf(false) }
    DevToolsBroadcastListener(onTrigger = { showDialog = true })
    content()
    if (showDialog) EnvSelectorDialog(onDismiss = { showDialog = false })
}
```

For **mobile + TV projects**, lift the reusable pieces (`DevToolsBroadcastListener`, `EnvSelectorDialog`, `ShakeListener`) into a `core/ui-base/` module (sibling to `core/designsystem-base`) so both `core/ui-mobile` and `core/ui-tv` can depend on it without `ui-tv` reaching across into `ui-mobile`. Mobile + TV `DevToolsHost.kt` files then live in their respective ui modules and pull the shared composables from `core/ui-base`.

For **mobile-only projects**, drop the `-base` split entirely and keep everything under `core/ui-mobile/dev/` — no extra module is worth the noise.

## Wiring at the app layer

`MainActivity` (or the closest equivalent) wraps content in `DevToolsHost(enabled = BuildConfig.IS_QA)`:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            <Project>Theme {
                DevToolsHost(enabled = BuildConfig.IS_QA) {
                    <Project>NavHost()
                }
            }
        }
    }
}
```

Same shape in `app-tv` using the TV variant of `DevToolsHost`. `BuildConfig.IS_QA` is set by the flavors convention plugin and is `false` on prod — so the host renders `content()` directly with zero overhead on shipped builds.

`<Project>Application` must include `environmentModule` in the Koin module list so `EnvSelectorDialog`'s `koinInject<EnvironmentConfig>()` resolves.

## Triggering the dialog

- **Phone**: shake the device. On the Android emulator, the simplest path is the broadcast command below — emulator virtual sensors are awkward to trigger reliably.
- **Phone or TV (CLI)**: `adb shell am broadcast -a <applicationId>.OPEN_DEV_TOOLS -p <applicationId>` (the `-p` scopes the broadcast to this app so other listeners on the device don't see it).
- **TV**: only the broadcast trigger works.

The wizard writes both options into the generated project's README under a "Development tools" section so the user can find them on day one.
