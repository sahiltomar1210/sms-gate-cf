# sms-gate-cf — Cloudflare Tunnel integration spec

> Fork of [capcom6/android-sms-gateway](https://github.com/capcom6/android-sms-gateway) that bundles Cloudflare Tunnel orchestration directly into the Android app. Zero third-party servers, zero rooted-device requirement, one-tap setup.

## 1. Problem we're solving

Upstream sms-gate.app has three operating modes:

| Mode | How outside world reaches the phone |
|---|---|
| Local only | Caller is on the same WiFi / LAN |
| Cloud managed | Phone outbound-polls `api.sms-gate.app` broker (200 msg/month free, then paid) |
| Private server | User self-hosts the broker on a VPS |

Nothing in-app for arbitrary callers over the internet without either a broker dependency or manual Cloudflare Tunnel setup via CLI. Our fork adds a fourth mode:

**Cloudflare Tunnel (new)** — one-tap setup inside the app. Caller gets a public HTTPS URL that routes directly to the phone's local Ktor server. No broker, no VPS, no CLI, no root.

## 2. Decisions already locked

| Decision | Choice | Rationale |
|---|---|---|
| Strategy | Fork + extend upstream | Keeps us in sync with capcom6's SMS core; our code is additive |
| Target root state | Non-rooted default; rooted = optional "Enhanced mode" | Addressable device pool ≈ 100% of Android ≥ API 26 |
| cloudflared binary | Downloaded on first setup, not shipped in APK | APK stays small (~5MB vs ~55MB); binary verified via SHA-256 |
| Cloudflare auth | Browser-based login (same flow as `cloudflared tunnel login`) | Slickest UX; no Cloudflare OAuth2 provider exists so we mimic the CLI's cert.pem issuance |
| v1 feature scope | Quick Tunnel **and** Named Tunnel from day one | Quick for demos; Named for prod |
| Battery / background | Foreground service + `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` + per-OEM wizard + boot receiver | Target 95%+ uptime on non-rooted |
| Fork visibility | Public (Apache-2 inherited) | Code is useless without the user's own account; publishing fuels community |

## 3. Architecture

New Kotlin package: `me.capcom.smsgateway.modules.cloudflare`

```
app/src/main/java/me/capcom/smsgateway/modules/cloudflare/
├── CloudflareTunnelService.kt       — orchestrator: lifecycle, state machine, calls child modules
├── CloudflareTunnelSettings.kt      — persisted preferences (mode, URL, enabled, cert?)
├── TunnelMode.kt                    — enum: Disabled, QuickTunnel, NamedTunnel
├── TunnelState.kt                   — sealed: Stopped, Starting, Connected(url), Error(msg)
├── binary/
│   ├── CloudflaredBinaryManager.kt  — download, verify SHA-256, extract, chmod +x
│   └── BinaryUrls.kt                — Cloudflare release URLs per ABI
├── process/
│   ├── CloudflaredProcess.kt        — ProcessBuilder wrapper, stdout parsing, stderr capture
│   └── QuickTunnelParser.kt         — regex-extracts trycloudflare.com URL from logs
├── auth/
│   ├── CloudflareLoginFlow.kt       — mimics `cloudflared tunnel login` (Custom Tab + callback)
│   ├── OriginCertStore.kt           — EncryptedSharedPreferences for cert.pem
│   └── CallbackReceiver.kt          — tiny HTTP server (NanoHTTPD) on loopback for cert pickup
├── api/
│   ├── CloudflareApiClient.kt       — OkHttp wrapper for api.cloudflare.com/client/v4
│   ├── ZonesRepo.kt                 — GET /zones (list user's domains)
│   ├── TunnelsRepo.kt               — POST /accounts/{id}/cfd_tunnel (create tunnel)
│   └── DnsRepo.kt                   — POST /zones/{id}/dns_records (CNAME to tunnel)
├── service/
│   ├── TunnelForegroundService.kt   — persistent notification, starts cloudflared process
│   └── BootReceiver.kt              — BOOT_COMPLETED → re-launch service if enabled
└── root/
    ├── RootDetector.kt              — detects su/magisk/KernelSU
    └── KsuModuleInstaller.kt        — v2 only: optionally install a KSU module for Enhanced mode
```

### Integration seams in upstream code (lightest-touch)

| Upstream file | Change |
|---|---|
| `MainActivity.kt` | No change |
| `ui/SettingsFragment.kt` | +1 PreferenceCategory linking to our fragment |
| `ui/settings/` | +1 new fragment: `CloudflareTunnelSettingsFragment.kt` |
| `res/xml/root_preferences.xml` | +1 `<Preference>` entry |
| `res/xml/cloudflare_tunnel_preferences.xml` | **new** file |
| `App.kt` (Koin modules) | +1 module registration: `cloudflareModule` |
| `modules/localserver/` | **no change** — our tunnel attaches to the port it already binds |
| `AndroidManifest.xml` | +permissions (see §5), +service + receiver declarations |
| `build.gradle` | +deps: `nanohttpd:nanohttpd:2.3.1`, `androidx.security:security-crypto:1.1.0-alpha06` |

Nothing in upstream's SMS/HTTP/webhook code is touched. All our files live in a new package. Upstream rebases cleanly.

## 4. UX flow

### First-time setup

```
Open app → onboarding → "Tunnel mode" screen with three cards:
  1. Local only (default)              — current behavior
  2. Quick Tunnel (trycloudflare.com)  — anonymous, free, ephemeral URL
  3. Named Tunnel (sms.your.domain)    — needs Cloudflare account
```

### Quick Tunnel path

```
Tap "Quick Tunnel"
  ↓
"We'll download the Cloudflare tunnel binary (~40MB)" → confirm
  ↓
Download + verify SHA-256 (progress bar)
  ↓
"Battery permission" screen → tap → OS exemption dialog → granted
  ↓
"Keep app running" per-OEM screen (see OEM_BATTERY.md) → tap → opens vendor-specific settings
  ↓
Start tunnel → cloudflared child process launches
  ↓
Parse stdout for https://*-*-*.trycloudflare.com URL
  ↓
Show URL with QR + copy button → done
```

### Named Tunnel path

```
Tap "Named Tunnel" → "Connect Cloudflare account"
  ↓
Chrome Custom Tab opens https://dash.cloudflare.com/argotunnel?aud=&callback=http://127.0.0.1:<port>/cf-callback
  ↓
App's embedded NanoHTTPD on 127.0.0.1:<port> waits for the cert.pem POST
  ↓
User authorizes in browser → Cloudflare POSTs cert.pem to the local callback
  ↓
Store cert.pem in EncryptedSharedPreferences
  ↓
Call GET /zones with Origin-Cert auth → show picker: [becholiya.com] [example.org]
  ↓
User picks a zone → prompt for subdomain (default "sms")
  ↓
Call POST /accounts/{id}/cfd_tunnel → tunnel ID + credentials
  ↓
Call POST /zones/{id}/dns_records → CNAME <sub>.zone → <uuid>.cfargotunnel.com
  ↓
Write tunnel credentials to app's internal storage
  ↓
Battery permission + OEM wizard (same as Quick Tunnel)
  ↓
Start tunnel → shows https://sms.becholiya.com + copy button → done
```

### Status screen (always visible after setup)

```
┌────────────────────────────────────────┐
│ Tunnel: Named (sms.becholiya.com)      │
│ Status: ● Connected  (12h 34m uptime)  │
│ Edge POP: BOM        Protocol: QUIC    │
│                                         │
│ [ Copy URL ]  [ View logs ]  [ Pause ] │
│                                         │
│ ⚙ Disconnect     ⚠ Switch to Quick     │
└────────────────────────────────────────┘
```

## 5. Permissions

See [PERMISSIONS.md](./PERMISSIONS.md) for the complete manifest + runtime request matrix.

High-level:
- `INTERNET` — tunnel outbound
- `ACCESS_NETWORK_STATE` — detect connectivity changes, restart tunnel
- `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_DATA_SYNC` (API 34+)
- `POST_NOTIFICATIONS` (API 33+) — persistent notification
- `RECEIVE_BOOT_COMPLETED` — re-launch tunnel after reboot
- `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` — user grant, keeps us out of doze
- `WAKE_LOCK` — hold partial wake lock during tunnel startup
- Upstream's existing: `SEND_SMS`, `READ_PHONE_STATE`, `RECEIVE_SMS`, etc. (not our concern)

## 6. Phase plan

| # | Phase | Files touched | Hours |
|---|---|---|---|
| 1 | Scaffold module skeleton + manifest + Koin wiring | ~15 new files, empty bodies | 2 |
| 2 | PreferenceFragment UI + root preferences entry | 3 files | 1.5 |
| 3 | CloudflaredBinaryManager (download, verify, extract) | 3 files | 2 |
| 4 | TunnelForegroundService + CloudflaredProcess (Quick Tunnel E2E) | 4 files | 4 |
| 5 | Battery exemption wizard + per-OEM router | 2 files + per-OEM strings | 2 |
| 6 | OAuth-style login flow (Custom Tab + NanoHTTPD callback) | 3 files | 4 |
| 7 | Cloudflare API client + zone picker + Named Tunnel orchestration | 5 files | 5 |
| 8 | Status screen + copy URL + QR + disconnect | 2 files | 2 |
| 9 | Boot receiver + crash restart | 2 files | 1.5 |
| 10 | Encrypted secret storage (cert.pem, tunnel creds) | 1 file | 1 |
| 11 | Polish, i18n (en + hi + mr matching Krushi Bharat), release signing | — | 3 |
| 12 | Optional: KSU module auto-install for Enhanced mode | 3 files | 3 |

**Total**: ~31 hours including Enhanced mode, ~28 without.

Realistic cadence: 3 focused sessions, each ~10 hours. Session 1 = phases 1-5 (working Quick Tunnel APK). Session 2 = phases 6-10 (Named Tunnel + full lifecycle). Session 3 = phase 11-12 + release.

## 7. API endpoints we'll call

Base: `https://api.cloudflare.com/client/v4`

| Call | Purpose | Auth |
|---|---|---|
| `GET /zones` | List user's domains for the picker | Origin cert |
| `POST /accounts/{aid}/cfd_tunnel` | Create tunnel, returns UUID + secret | Origin cert |
| `POST /zones/{zid}/dns_records` | Create CNAME `<sub>.zone → <uuid>.cfargotunnel.com` | Origin cert |
| `GET /accounts/{aid}/cfd_tunnel/{tid}/connections` | Health probe for status screen | Origin cert |
| `DELETE /accounts/{aid}/cfd_tunnel/{tid}` | Disconnect cleanup | Origin cert |

Login URL (not an API, but the browser flow):
```
https://dash.cloudflare.com/argotunnel?aud=&callback=http://127.0.0.1:<port>/cf-callback
```
Cloudflare's dashboard handles the "pick an account + zone, authorize" UI itself, then POSTs the signed cert.pem to our callback.

## 8. Secret storage

| Secret | Storage | Why |
|---|---|---|
| `cert.pem` (origin cert) | `EncryptedSharedPreferences` (AES-256-GCM, keystore-backed) | Needed per-launch for API calls; must not plaintext-leak |
| Tunnel credentials JSON | Same | Needed by cloudflared on every startup |
| Phone HTTP server Basic Auth password | Already in SharedPreferences (upstream) — we don't touch |

Android keystore keys bind to the app's package signature → extracting via adb pull doesn't give usable plaintext even on rooted devices (barring full keystore extraction, which needs TEE compromise).

## 9. Error handling + resilience

- **Download failure** → retry 3x with backoff; cache partial download; show user error with "Retry" button
- **Checksum mismatch** → refuse to run binary, show error (likely MITM or truncated download)
- **cloudflared crash** → service.sh-equivalent in Kotlin: watch exit code, restart with exponential backoff capped at 60s
- **Network loss (WiFi off, mobile data drop)** → cloudflared auto-reconnects (its own retry loop); we just monitor
- **App killed by OEM** → boot receiver + `AlarmManager` fallback: if we detect we were killed in the last hour, show a notification suggesting the OEM wizard again
- **Cert expired / revoked** → API calls 401 → prompt user to re-login
- **DNS already exists** on named tunnel setup → offer "Overwrite" or "Pick different subdomain"

## 10. Non-goals (explicitly out of scope for v1)

- Multi-tunnel per phone (one tunnel per device)
- SIM slot picker inside tunnel settings (unrelated to tunnel; upstream handles)
- Tunnel sharing / team workflow
- Metrics / analytics beyond what cloudflared reports
- Web dashboard
- iOS version (Capacitor plugin someday maybe)
- Running cloudflared without a persistent notification (Android policy says foreground service MUST show notification)

## 11. Testing matrix

Target devices for pre-release testing:
- Samsung Galaxy A54 (One UI 6, Android 14) — our reference device
- Xiaomi Redmi (MIUI) — aggressive battery killer
- Pixel 7 (Vanilla Android 14) — reference for AOSP behavior
- OnePlus (OxygenOS) — moderate battery killer
- Android 10 device — minimum supported API 29

Scenarios:
1. Fresh install → Quick Tunnel setup → send SMS via public URL → works
2. Fresh install → Named Tunnel setup → send SMS → works
3. Reboot phone → tunnel auto-restarts within 2 min → send SMS → works
4. Toggle WiFi off → mobile data takes over → tunnel reconnects → works
5. Kill app from recents → tunnel service survives (fg service) OR AlarmManager restarts within 15 min
6. Device idle for 8h with screen off → tunnel still serves requests (ignore_battery_optimizations granted)
7. Rooted device → auto-detect root → offer Enhanced mode

## 12. Release checklist

- [ ] Debug APK on 5+ test devices
- [ ] Battery drain test: < 2% per 24h in idle (foreground service with cloudflared ~90% idle)
- [ ] Release signing with reproducible key (store password in GitHub Actions secret for CI builds)
- [ ] GitHub Release with signed APK + CHANGELOG
- [ ] Update sms-otp-stack README to reference sms-gate-cf as the recommended phone-side app
- [ ] Mention the fork in a PR to upstream capcom6 as "related project" (not asking for merge; just discoverability)
