# OEM battery killer wiring

Non-rooted Android foreground services survive on stock AOSP (Pixel) indefinitely once `IGNORE_BATTERY_OPTIMIZATIONS` is granted. Every major Chinese and Korean OEM adds their own layer on top that ignores Google's whitelist and still kills "unused" background apps. We can't programmatically override these — users must flip vendor switches manually. The setup wizard detects `Build.MANUFACTURER` and shows the right instructions.

## Detection

```kotlin
val oem = Build.MANUFACTURER.lowercase()
val wizard = when {
    oem == "samsung" -> SamsungWizard
    oem == "xiaomi" || oem == "redmi" || oem == "poco" -> XiaomiWizard
    oem == "oppo" || oem == "realme" -> OppoWizard
    oem == "vivo" || oem == "iqoo" -> VivoWizard
    oem == "huawei" || oem == "honor" -> HuaweiWizard
    oem == "oneplus" -> OnePlusWizard
    oem == "google" || oem == "nothing" || oem == "asus" || oem == "motorola" -> StockWizard
    else -> GenericWizard
}
```

## Per-vendor instructions

### Samsung (One UI 5+)

Battery killer: aggressive. Services survive ~1-3h after screen off without the setting.

**Fix sequence:**
1. Settings → **Battery** → **Background usage limits**
2. **Never sleeping apps** → **+** → tick **SMS Gateway**
3. Also disable **Put unused apps to sleep** (same screen)

**Deep link (works on One UI 5+):**
```kotlin
val intent = Intent("com.samsung.android.sm.ACTION_BATTERY").apply {
    setPackage("com.samsung.android.lool")
}
startActivity(intent)
```
Fallback: `Settings.ACTION_APPLICATION_DETAILS_SETTINGS`

### Xiaomi / Redmi / POCO (MIUI 12+)

Battery killer: most aggressive on the market. Multiple layers.

**Fix sequence:**
1. Settings → **Battery** → **App battery saver** → tap **SMS Gateway** → **No restrictions**
2. Settings → **Apps** → **Manage apps** → **SMS Gateway** → **Autostart** → **ON**
3. Settings → **Apps** → **Manage apps** → **SMS Gateway** → **Other permissions** → **Display pop-up windows while running in background** → **ON**
4. In recent apps, long-press SMS Gateway card → **Lock** (padlock icon)

**Deep link (MIUI 12+):**
```kotlin
val intent = Intent("miui.intent.action.POWER_MANAGER_SETTINGS")
```
Autostart deep link:
```kotlin
val intent = Intent().setComponent(ComponentName(
    "com.miui.securitycenter",
    "com.miui.permcenter.autostart.AutoStartManagementActivity"
))
```

MIUI's "App Vault" and "Dual Apps" features can also break foreground services — note in the wizard.

### Oppo / Realme (ColorOS 12+)

**Fix sequence:**
1. Settings → **Battery** → **Battery optimization** → **SMS Gateway** → **Don't optimize**
2. Settings → **Battery** → **Background power consumption management** → **SMS Gateway** → **Allow background activity**
3. Settings → **Apps and permissions** → **App startup manager** → **SMS Gateway** → enable all three toggles (Auto-launch, Secondary launch, Run in background)

**Deep link:**
```kotlin
val intent = Intent().setComponent(ComponentName(
    "com.coloros.safecenter",
    "com.coloros.safecenter.startupapp.StartupAppListActivity"
))
```

### Vivo / iQOO (Funtouch OS / Origin OS)

**Fix sequence:**
1. iManager (app) → **App manager** → **Autostart manager** → enable SMS Gateway
2. Settings → **Battery** → **Background power consumption** → SMS Gateway → **Allow**
3. Settings → **Battery** → **High-background power consumption** → SMS Gateway → **ON**

**Deep link:**
```kotlin
val intent = Intent().setComponent(ComponentName(
    "com.iqoo.secure",
    "com.iqoo.secure.ui.phoneoptimize.AddWhiteListActivity"
))
```

### Huawei / Honor (EMUI 10+ / MagicOS)

Phone Manager app controls everything. Google services often absent so our FGS behavior differs.

**Fix sequence:**
1. Phone Manager → **Protected apps** → SMS Gateway → **ON**
2. Settings → **Battery** → **App launch** → SMS Gateway → **Manage manually** → enable all three
3. Settings → **Apps** → **SMS Gateway** → **Power usage details** → enable **Keep running after screen off**

**Deep link:**
```kotlin
val intent = Intent().setComponent(ComponentName(
    "com.huawei.systemmanager",
    "com.huawei.systemmanager.optimize.process.ProtectActivity"
))
```

### OnePlus (OxygenOS 11+)

Moderate killer. Usually just the stock battery exemption is enough; the backup setting is:

**Fix sequence:**
1. Settings → **Battery** → **Battery optimization** → SMS Gateway → **Don't optimize** (same as stock)
2. Settings → **Battery** → **Advanced optimization** → **Deep optimization** → **OFF** for SMS Gateway

### Pixel / Nothing / ASUS / Motorola (stock AOSP)

Battery exemption via standard `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` is sufficient. No extra OEM steps needed. Wizard shows:
> "Your device uses standard Android power management. The battery exemption you granted is all that's needed."

### Generic / unknown OEM

Fallback: open the standard battery optimization settings and link to [dontkillmyapp.com](https://dontkillmyapp.com) for the user's specific brand.

## In-app wizard UI

Each vendor screen has:
- Vendor logo + name banner
- Numbered steps with screenshots (bundle one screenshot per step per vendor)
- "Open settings" button with the deep link
- "I've done this" / "Skip for now" / "Having trouble?" buttons
- Link to "View all vendors" if auto-detection was wrong

## What happens on denial / skip

Status screen shows a tiered health badge:

- 🟢 **Protected** — Battery exemption + OEM exceptions set
- 🟡 **Degraded** — Battery exemption OK, OEM wizard skipped
- 🔴 **Standby** — Nothing granted; tunnel works only while app is foreground

The badge is tappable → re-opens the wizard for whichever step is missing.

## Validation

Post-wizard, re-check and surface the result:

```kotlin
val pm = context.getSystemService(PowerManager::class.java)
val battery = pm.isIgnoringBatteryOptimizations(context.packageName)
```

For OEM-specific state, there's no reliable API — we just show the badge based on manifest-detectable state + a user self-report checkbox they ticked ("I've done the OEM steps").

## Reference

- [dontkillmyapp.com](https://dontkillmyapp.com) — community-maintained OEM behavior database
- [AOSP: foreground services](https://developer.android.com/guide/components/foreground-services) — Google's canonical reference
- [Android 14 FGS restrictions](https://developer.android.com/about/versions/14/changes/fgs-types-required) — mandatory `foregroundServiceType` for API 34+
