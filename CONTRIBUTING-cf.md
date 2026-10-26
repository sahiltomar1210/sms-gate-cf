# Contributing to the Cloudflare integration

How to work on sms-gate-cf's `modules/cloudflare/` without disrupting upstream capcom6/android-sms-gateway. Rules + conventions for keeping the fork rebase-clean.

## Golden rule

**Our code lives in `me.capcom.smsgateway.modules.cloudflare`. Touch upstream files only via the integration seams listed in [SPEC.md §3](./SPEC.md#integration-seams-in-upstream-code-lightest-touch).**

Any file outside `modules/cloudflare/` that we modify is a merge conflict waiting to happen when we rebase on upstream. Keep those modifications tiny, well-marked (`// cf-fork: added`), and listed in SPEC.

## Branches

- `master` — tracks upstream `capcom6/android-sms-gateway:master`, synced via `gh repo sync`
- `cf-main` — our integration lives here; always PR-merged from feature branches
- `cf/phase-N-<thing>` — short-lived feature branches per phase

Release tags: `v1.0.0-cf.1`, `v1.0.0-cf.2`, …

## Upstream sync workflow

```bash
cd ~/Documents/Codex/2026-09-09/sms-gate-cf
gh repo sync sahiltomar1210/sms-gate-cf --source capcom6/android-sms-gateway --branch master
git fetch origin
git checkout cf-main
git rebase origin/master
# Resolve conflicts (should only be in files we marked with cf-fork comments)
git push --force-with-lease
```

If the rebase gets ugly, inspect what upstream changed near our integration seams — most upstream churn is in `modules/messages/` and `modules/mms/` which we never touch.

## Picking up where SPEC left off

Each phase in SPEC.md has a "Files touched" column. Start a session by:

1. `git checkout cf-main && git pull`
2. `git checkout -b cf/phase-N-<thing>`
3. Create the files listed for that phase
4. Reference SPEC.md section for the behavior spec
5. Commit, push, open a PR against `cf-main`

## Testing on-device

```bash
# After phase 1 completes (compiles but no-ops):
cd ~/Documents/Codex/2026-09-09/sms-gate-cf
export JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell monkey -p me.capcom.smsgateway -c android.intent.category.LAUNCHER 1
```

After phase 4 (first functional Quick Tunnel), end-to-end test on reference device (Samsung A54):
1. Open app → enable Tunnel → Quick Tunnel
2. Grant all permissions in wizard
3. Verify public URL appears
4. From another device: `curl https://<url>.trycloudflare.com/health -u sms:<pass>`
5. Should return the gateway's health JSON

## Code style

- Match upstream's existing Kotlin conventions (4-space indent, trailing commas, no wildcard imports)
- `.editorconfig` is inherited from upstream — don't change
- Public APIs documented with KDoc; internal helpers can skip docs
- Use Koin for DI, matching upstream pattern (`by inject<X>()` in Activities/Services, module declarations in `App.kt`)
- Coroutines over Thread/Executor; `CoroutineScope(Dispatchers.IO)` for network, `Dispatchers.Default` for CPU
- No new third-party dependencies without discussion. Current approved list:
  - `nanohttpd` (callback server for OAuth-style flow)
  - `androidx.security:security-crypto` (EncryptedSharedPreferences)
  - OkHttp (already in upstream)

## Secret handling

- **Never** commit secrets, test tokens, or real tunnel UUIDs
- EncryptedSharedPreferences for cert.pem and tunnel credentials
- `android:allowBackup="false"` already set by upstream — keep it
- Debug builds must not log secrets. Prod builds use `Timber` with release-tree stripping

## PR checklist

- [ ] Phase N complete per SPEC.md checklist
- [ ] Changes outside `modules/cloudflare/` are listed in PR description with justification
- [ ] On-device test on at least one physical device
- [ ] No new permissions added without updating PERMISSIONS.md
- [ ] If touching UI: screenshot in PR description
- [ ] i18n: new strings added to `values/strings.xml` AND `values-hi/strings.xml` + `values-mr/strings.xml`
