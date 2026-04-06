---
id: v-wmoe
status: done
deps: []
links: []
created: 2026-04-06T00:23:35Z
type: feature
priority: 2
assignee: Jibles
---
# Add Android emulator support to Detox E2E tests

## Objective

The Detox E2E test suite currently only supports iOS simulators. Add Android emulator support so the same tests can run on both platforms. The existing tests are 90% platform-agnostic (standard Detox APIs) — only the configuration and one helper need changes.

## Context & Findings

- `.detoxrc.js` is iOS-only: hardcoded `xcodebuild`, iOS simulator resolution via `xcrun simctl`, Xcode DerivedData paths
- All 5 test files (`e2e2e.test.js`, `balances.test.js`, `conversation.test.js`, `seedPhraseImport.test.js`, `vaultCreate.test.js`) use only generic Detox APIs (`by.id`, `tap`, `type`, `waitFor`) — zero platform-specific selectors
- All flow files (`setupViaCreate.js`, `setupViaImport.js`, `setupViaSeedPhrase.js`) and `e2e/helpers/common.js` are platform-agnostic
- **One iOS-coupled helper**: `e2e/helpers/importVault.js` line 23 uses `xcrun simctl get_app_container` to stage `.vult` files into the simulator. Android equivalent is `adb push` to the app's external files directory or `/sdcard/Download/`
- Available Android emulators on dev machine: `Medium_Phone_API_36.0`, `test_api34`, `test_api34_noplay`
- Android APK builds via `npx expo run:android` — the APK path is typically `android/app/build/outputs/apk/debug/app-debug.apk`
- DKLS AAR files (required for Android build) are already present at `modules/expo-dkls/android/libs/`
- Rejected: adapter layer / test abstraction — unnecessary since tests themselves are already platform-agnostic

## Files

- `vultiagent-app/.detoxrc.js` — add `android.emu.debug` app config, Android device config, and `android.emu.debug` configuration entry
- `vultiagent-app/e2e/helpers/importVault.js` — make `getAppContainerDirectory()` and `stageImportVaultFileForDetox()` platform-aware (branch on `device.getPlatform()` or env var)
- `vultiagent-app/package.json` — add `test:e2e:android` and `test:e2e:build:android` scripts
- `vultiagent-app/.env.example` — document any new Android-specific env vars (e.g. `DETOX_ANDROID_AVD`)

## Acceptance Criteria

- [x] `.detoxrc.js` has an `android.emu.debug` configuration alongside existing `ios.sim.debug`
- [x] Android app config uses Gradle build command and resolves APK path
- [x] Android device config resolves emulator AVD (via `DETOX_ANDROID_AVD` env or sensible default)
- [x] `importVault.js` stages vault files on Android via `adb push` (to app documents or `/sdcard/Download/`)
- [x] `package.json` has `test:e2e:android` and `test:e2e:build:android` scripts
- [ ] At least one existing test (`conversation.test.js` or `vaultCreate.test.js`) passes on an Android emulator
- [x] iOS tests still work unchanged (`npm run test:e2e` still targets iOS)
- [x] `.env.example` documents new Android env vars

## Gotchas

- Detox Android requires the app to be a debug build with `android:usesCleartextTraffic="true"` for localhost backend — check `AndroidManifest.xml`
- `device.clearKeychain()` works differently on Android — Detox maps it to clearing app data, but verify it doesn't wipe the staged vault file
- The Android bundle ID is `com.anonymous.vultiagentpoc` (same as iOS) — confirm this matches the `applicationId` in `android/app/build.gradle`
- Vault file staging on Android: if using `adb push` to `/sdcard/Download/`, the app's file picker must be able to read from there (check `expo-document-picker` permissions)
- The `runZsh()` helper in `common.js` shells out via zsh — works on Linux but verify the env vars propagate correctly in the Detox Android test harness
