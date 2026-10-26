# PERMISSIONS

Complete permission matrix for sms-gate-cf. Lists every manifest-declared permission (new to us or inherited from upstream) and every runtime grant the app needs, with the exact user flow.

## Our new manifest-declared permissions

```xml
<!-- Added by cloudflare integration -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC"
    tools:targetApi="34" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"
    tools:targetApi="33" />
<uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"
    tools:targetApi="31" />
```

| Permission | Why we need it | API min | User-prompted? |
|---|---|---|---|
| `INTERNET` | cloudflared outbound to CF edge | — | ❌ auto-granted |
| `ACCESS_NETWORK_STATE` | detect WiFi↔mobile data switches, restart on disconnect | — | ❌ auto-granted |
| `WAKE_LOCK` | partial wake lock during tunnel startup to avoid CPU sleep mid-handshake | — | ❌ auto-granted |
| `RECEIVE_BOOT_COMPLETED` | auto-start tunnel after device reboot | — | ❌ auto-granted (declared only; no runtime check) |
| `FOREGROUND_SERVICE` | run `TunnelForegroundService` with persistent notification | 28+ | ❌ auto-granted |
| `FOREGROUND_SERVICE_DATA_SYNC` | Android 14 requires a typed FGS category; tunnel is best-match "dataSync" | 34+ | ❌ auto-granted |
| `POST_NOTIFICATIONS` | persistent FGS notification needs this on API 33+ | 33+ | ✅ runtime prompt |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | opens the OS whitelist dialog | 23+ | ✅ system dialog |
| `SCHEDULE_EXACT_ALARM` | AlarmManager restart fallback if FGS gets killed | 31+ | ✅ user flips a Settings switch (not a dialog) |

## Upstream permissions we inherit (not our concern)

`SEND_SMS`, `READ_SMS`, `RECEIVE_SMS`, `READ_PHONE_STATE`, `RECEIVE_WAP_PUSH`, etc. — these are capcom6's business, not touched by our fork.

## Runtime grants — ordered by setup wizard

The setup wizard shows these one at a time with a brief rationale. User must tap through each. Any denial is non-fatal but logged, and the status screen later shows a yellow warning badge with "fix this to improve reliability".

### Step 1 — POST_NOTIFICATIONS (API 33+)

```kotlin
ActivityCompat.requestPermissions(
    activity,
    arrayOf(Manifest.permission.POST_NOTIFICATIONS),
    REQ_NOTIFICATIONS,
)
```

**Rationale shown to user**:
> "The tunnel runs as a background service. Android requires us to show a notification icon while it's running — this is non-removable per Google policy. The notification doesn't make sound or vibrate."

If denied: foreground service still works (API 33 downgrade — notification just hidden from the user, but service runs). We retry at next launch.

### Step 2 — REQUEST_IGNORE_BATTERY_OPTIMIZATIONS

```kotlin
val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
    data = Uri.parse("package:${context.packageName}")
}
startActivity(intent)
```

**Rationale shown to user**:
> "Android pauses background apps after a few minutes of screen-off to save battery. To keep SMS flowing 24/7, we need to be exempt. Grant now?"

If denied: app enters "degraded mode" — expect kills after ~30 min idle on stock Android, sooner on Samsung/Xiaomi. Status screen shows red badge "Tunnel will drop when screen is off."

### Step 3 — SCHEDULE_EXACT_ALARM (API 31+)

```kotlin
val am = context.getSystemService(AlarmManager::class.java)
if (!am.canScheduleExactAlarms()) {
    val intent = Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM).apply {
        data = Uri.parse("package:${context.packageName}")
    }
    startActivity(intent)
}
```

**Rationale**:
> "If the OS kills our service unexpectedly, we schedule an exact alarm to wake up and restart within 15 min. Without this permission we can only use inexact alarms (±1 hour)."

If denied: fallback to inexact AlarmManager, which still works but with worse resumption latency.

### Step 4 — OEM-specific "keep in memory" setting (manual, per vendor)

See [OEM_BATTERY.md](./OEM_BATTERY.md) for the full per-vendor matrix. Setup wizard detects the OEM via `Build.MANUFACTURER` and routes to the right vendor-specific how-to screen. Example for Samsung:

> **Samsung** devices put apps to sleep aggressively. To keep SMS running:
> 1. Open **Settings → Battery → Background usage limits**
> 2. **Never sleeping apps** → **Add apps** → tick **SMS Gateway**
> 3. Come back here and tap "I've done this"

The wizard can't programmatically do this (vendor settings are closed). Best we do is show the vendor's icon and open the relevant Settings deep-link if the OEM provides one (Samsung does via `com.samsung.android.lool`; MIUI blocks everything).

### Step 5 — (Rooted only, auto-detected) Grant Shell root to app

If `RootDetector.isRooted()` returns true, offer:

> "Your device is rooted. For best reliability we can run the tunnel as a system service that survives even if the app is swiped from recents. Tap to install the companion KernelSU/Magisk module."

Not v1 priority. See phase 12 in [SPEC.md](./SPEC.md#6-phase-plan).

## Permission denial resilience

The whole wizard is skippable. If the user denies everything:
- Tunnel still works while the app is in the foreground
- Dies when screen off or app swiped from recents
- Status screen says "Standby mode — launch the app to resume"

This is a legitimate low-impact deployment mode (e.g., user runs this on their primary phone, launches once per hour to batch-send) and should remain supported.

## What we DO NOT ask for

- `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` — not needed, binary lives in `nativeLibraryDir` + settings in `EncryptedSharedPreferences`
- `READ_PHONE_NUMBERS` — the tunnel doesn't care who the phone is
- `ACCESS_FINE_LOCATION` — no reason
- `CAMERA` — unless we add QR scanning later (not in v1)
- `ACCESS_WIFI_STATE` — `ACCESS_NETWORK_STATE` covers what we need
- `BIND_ACCESSIBILITY_SERVICE` — we don't need to monitor other apps
- `SYSTEM_ALERT_WINDOW` — no overlays

## Battery-impact estimate

With foreground service + cloudflared idle (no SMS being sent):
- CPU: ~0% average (cloudflared is QUIC-idle, event-driven)
- Network: ~5 KB/min keepalive to Cloudflare edge
- Battery drain: ≈ 1-2% per 24h on a Galaxy A54 (observed in our KSU deployment)

Burst during SMS send: negligible — one HTTP POST through the tunnel + one SmsManager call. Measured impact: < 0.1% per 100 SMS.

## Privacy notes

- No analytics, no crash reporting, no telemetry
- All secrets (tunnel creds, cert.pem, HTTP basic auth) stay on-device in EncryptedSharedPreferences
- No data is sent to anyone except: Cloudflare (tunnel ingress), the user's own callers (SMS content), the carrier (SMS content)
- If the user picks a Named Tunnel, their Cloudflare account email + zone list are visible to our Kotlin code during setup — never sent to anyone else
