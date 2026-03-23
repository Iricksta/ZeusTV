# ZeusTV Changelog

## [Feature] Full Cross-Device Sync — Watch Progress + All Settings — 2026-03-23

### Phase 1 — Continue Watching Periodic Refresh
Fixed: watch progress from other devices now appears on TV/phone within ~90 seconds without needing to restart the app.

Root cause: Android TV keeps the app permanently foregrounded, so `onResume()` never fires after initial startup, meaning `requestSyncNow()` was never called again.

Fix: Added a 90-second periodic sync coroutine in `MainActivity` (`periodicWatchSyncJob`) that calls a new lightweight `requestWatchProgressSyncNow()` in `StartupSyncService`. This method only pulls watch progress + watched items (skips the heavier profile/addon/plugin/library sync).

**Files changed:**
- `app/.../core/sync/StartupSyncService.kt` — added `requestWatchProgressSyncNow()`
- `app/.../MainActivity.kt` — added `periodicWatchSyncJob` (start in `onResume`, cancel in `onPause`)

### Phase 2 — Full Settings Sync Per Profile
All user settings now sync to Supabase per-profile. Changes push within 2 seconds; pulls happen on app start and every 90 seconds.

**Settings groups synced:**
- `player` — PlayerSettingsDataStore (50+ keys: player pref, libass, decoder, subtitle style, buffer, autoplay, etc.)
- `layout` — LayoutPreferenceDataStore (25+ keys: layout, hero, sidebar, poster, backdrop trailer, etc.)
- `theme` — ThemeDataStore (selected theme, font)
- `trakt` — TraktSettingsDataStore (continue watching days cap, show unaired, watch progress source, dismissed next-up)
- `trailer` — TrailerSettingsDataStore (enabled, delay seconds)
- `anime_skip` — AnimeSkipSettingsDataStore (enabled, client ID)
- `track_prefs` — TrackPreferenceDataStore (per-content audio/subtitle track preferences)

**Not synced (intentional):**
- TraktAuthDataStore — auth tokens (security)
- AuthSessionNoticeDataStore — device-specific
- DebugSettingsDataStore — dev only
- AppOnboardingDataStore — device-specific
- StreamLinkCacheDataStore — ephemeral cache

**New files:**
- `app/.../data/local/PreferencesExtensions.kt` — `Preferences.toJsonObject()` utility
- `app/.../core/sync/SettingsSyncService.kt` — Supabase push/pull service
- `supabase/profile_settings_sync.sql` — SQL to run in Supabase dashboard

**Files changed:**
- `app/.../data/local/PlayerSettingsDataStore.kt`
- `app/.../data/local/LayoutPreferenceDataStore.kt`
- `app/.../data/local/ThemeDataStore.kt`
- `app/.../data/local/TraktSettingsDataStore.kt`
- `app/.../data/local/TrailerSettingsDataStore.kt`
- `app/.../data/local/AnimeSkipSettingsDataStore.kt`
- `app/.../data/local/TrackPreferenceDataStore.kt`
- `app/.../core/sync/StartupSyncService.kt`

### Action Required (User)
Run `supabase/profile_settings_sync.sql` in the Supabase SQL editor before testing.

---

## [Audit] Pre-Release Full Branding, Security & Performance Audit — 2026-03-23

### Summary
Completed a full pre-release audit across security, branding, performance, and asset cleanup. All phases executed.

### Phase 1 — Security
- **build.gradle.kts**: Removed hardcoded fallback keystore passwords (`Wsxqaz2411.`) — build now fails fast with a clear error if credentials are missing
- **AndroidManifest.xml**: Set `android:allowBackup="false"` to prevent ADB backup extraction of auth tokens
- **network_security_config.xml** (NEW): Created fine-grained network security config enforcing HTTPS on Zeus backend domains (supabase.co, vercel.app, themoviedb.org, trakt.tv, googleapis.com) while keeping cleartext enabled globally for addon/streaming CDN compatibility

### Phase 2 — User-Facing Branding
- **AddonWebPage.kt**: HTML `<title>` and logo `alt` updated from NuvioTV → Zeus
- **RepositoryWebPage.kt**: HTML `<title>` and logo `alt` updated from NuvioTV → Zeus
- **PluginManager.kt**: `User-Agent` header changed from `NuvioTV/1.0` → `Zeus/1.0` (sent to addon servers)
- **SidebarNavigation.kt**: App name text `"NUVIO"` → `"ZEUS"`
- **ZeusTopBar.kt**: App name text `"NUVIO"` → `"ZEUS"`
- **ModernSidebarBlurPanel.kt**: Content description `"NuvioTV"` → `"Zeus"`
- **ProfileSelectionScreen.kt**: Content description `"NuvioTV"` → `"Zeus"`
- **AboutScreen.kt**: Content description + privacy policy URL updated to Zeus repo
- **PlaybackSettingsSections.kt**: Description `"Use NuvioTV's built-in player"` → `"Use Zeus's built-in player"`
- **SettingsDesignSystem.kt**: Content description `"NuvioTV"` → `"Zeus"`
- **SupportersContributorsScreen.kt**: Content description `"NuvioTV"` → `"Zeus"`
- **PlayerRuntimeController.kt**: Internal subtitle track ID prefix `"nuvio-addon-sub:"` → `"zeus-addon-sub:"`
- **README.md**: Full rewrite — all NuvioTV/tapframe references replaced with ZeusTV/Iricksta
- **CONTRIBUTING.md**: Line 3 updated to ZeusTV

### Phase 3 — Performance
- **compose_stability_config.conf**: Fixed package names from `com.nuvio.tv.*` → `com.zeus.tv.*` — Compose compiler stability hints were being silently ignored

### Phase 4 — Internal Branding
- **GitHubContributorsRepository.kt**: OWNER `"tapframe"` → `"Iricksta"`, TV_REPOSITORY `"NuvioTV"` → `"ZeusTV"`, MOBILE_REPOSITORY updated
- **GitHubContributorsRepositoryTest.kt**: Mock data repo keys updated to match
- **NuvioScrollDefaults.kt → ZeusScrollDefaults.kt**: Object renamed, file renamed, MainActivity imports/usages updated
- **AuthSessionNoticeDataStore.kt**: Enum `StartupAuthNotice.NUVIO` → `ZEUS`, public functions renamed (`markZeusAuthenticated`, `markZeusExplicitLogout`, `markUnexpectedZeusLogoutIfNeeded`)
- **AuthManager.kt**: All call sites updated to new function names
- **HomeScreen.kt**: Enum branch `NUVIO` → `ZEUS`
- **TraktSettingsDataStore.kt**: `WatchProgressSource.NUVIO_SYNC` → `ZEUS_SYNC`
- **TraktScreen.kt**: All `NUVIO_SYNC` references updated to `ZEUS_SYNC`

### Phase 5 — Asset Cleanup
Deleted 35 obsolete/temporary files from `assets/`:
- Old Nuvio logo (`nuviotv.png`, `zeustv.png`)
- All banner preview iterations (`zeus_banner_preview*.png`)
- All ADB verification screenshots (`verify_*.png`, `current_state*.png`, `qr_*.png`)

**Kept:** `zeus_banner_final.png`, `assets/zeus/`, `assets/brand/`

### Files NOT Changed (Intentional)
- `NuvioColors`, `NuvioTheme`, `NuvioColorScheme`, `buildNuvioTypography` — internal theme class names, zero user-visible impact, deferred to post-v1.0 refactor
- DataStore preference string keys (`"had_nuvio_auth"`, etc.) — changing these would wipe existing user session data

---

## [Fix] QR Code Login — Anonymous User Trigger Bug — 2026-03-23

### Summary
QR login screen showed "QR unavailable. Refresh to retry." / "An unexpected error occurred." Root cause: the `block_non_whitelisted_signups` trigger on `auth.users` fired for ALL new users including anonymous ones. Anonymous users have NULL email, so the whitelist check crashed Supabase auth before the temporary QR session could be created.

### Fix Applied (Supabase SQL)
Updated `block_non_whitelisted_signups()` function to skip whitelist check when `NEW.email IS NULL`:
```sql
IF NEW.email IS NULL THEN
  RETURN NEW; -- Allow anonymous users (TV app QR login flow)
END IF;
```

### Verified
TV screenshot confirmed: QR code generates successfully, code displayed, "Waiting for approval on your phone."

### No app rebuild required — backend-only fix.

---

## [Branding] TV Banner Centred + Black Background — 2026-03-23

### Summary
Fixed two issues with the Zeus TV tile: (1) logo was offset to top-left instead of centred, (2) transparent background caused Google TV launcher to show its own olive/khaki tile colour. Fixed by regenerating banner with black (#000000) background and centred logo layout. Result: clean black tile with ZEUS centred, consistent with other installed apps.

### Files Changed
- `app/src/main/res/mipmap-xhdpi/banner.png` — 320×180 black background, centred Z+EUS
- `app/src/main/res/drawable/tv_banner.png` — Same
- `app/src/main/res/drawable-nodpi/tv_banner.png` — Same

### Source Asset
- `C:\ZeusTV\assets\zeus_banner_final.png`

### Banner Spec
- Canvas: 320×180 black fill
- Icon: 155×155, centred at x=59, y=12
- Font: Cinzel-Bold, size=54
- Overlap: 45px (29%)
- Text x=169, gradient orange (#f5a623) → purple (#8b5cf6)

---

## [Branding] TV Banner Z+EUS Final — 2026-03-23

### Summary
TV banner updated to show the full Zeus logo: Z icon + "EUS" in Cinzel-Bold with orange (#f5a623) → purple (#8b5cf6) horizontal gradient. Transparent background (no rectangle box on TV home screen). Generated from source files (`zeus_icon_transparent.png` + `Cinzel-Bold.ttf`) using Python/Pillow with 29% overlap between icon and text. Google TV launcher cache cleared with `pm clear com.google.android.apps.tv.launcherx` after each install to force banner reload.

### Files Changed
- `app/src/main/res/mipmap-xhdpi/banner.png` — Z+EUS banner (320×180, transparent)
- `app/src/main/res/drawable/tv_banner.png` — Same
- `app/src/main/res/drawable-nodpi/tv_banner.png` — Same

### Source Asset
- `C:\ZeusTV\assets\zeus_banner_preview4.png` — Approved banner source

### Key Learning
Google TV launcher aggressively caches banners — after every reinstall, run `adb shell pm clear com.google.android.apps.tv.launcherx` to force the launcher to reload banners from the newly installed APK.

---

## [Branding] Zeus Logo Final Fix — 2026-03-23

### Summary
Logo was captured directly from https://zeustv.vercel.app/ using Playwright (scaled to 5x, background removed, cropped). This guarantees a pixel-perfect match to the website logo. Previous Python-generated versions had incorrect positioning and appeared as "double Z" on TV.

### Files Changed
- `app/src/main/res/mipmap-xhdpi/banner.png` — Correct Zeus logo banner (320×180, transparent, from website)
- `app/src/main/res/drawable/tv_banner.png` — Same
- `app/src/main/res/drawable-nodpi/tv_banner.png` — Same
- `app/src/main/res/drawable/app_logo_wordmark.png` — Zeus wordmark (480×120, from website)
- `app/src/main/res/drawable/app_logo_mark.png` — Zeus icon only (1080×1080, transparent)

### Source Assets (saved to `C:\ZeusTV\assets\zeus\`)
- `tv_banner_320x180.png` — Final TV banner (website screenshot, bg removed)
- `wordmark_480x120.png` — Final wordmark (website screenshot, bg removed)
- `logo_mark_1080x1080.png` — Icon only
- `zeus_icon_transparent.png` — Source Z icon (512×512)
- `Cinzel-Bold.ttf` — Brand font

---

## [Branding] Zeus Logo Update — 2026-03-23

### Summary
Replaced all app logo and banner assets with the correct Zeus brand logo: the Z icon (`zeus_icon_transparent.png`) + "eus" in Cinzel-Bold with an orange (#f5a623) to purple (#8b5cf6) horizontal gradient. Assets now have transparent backgrounds (no dark box on TV home screen).

### Files Changed
- `app/src/main/res/mipmap-xhdpi/banner.png` — New Zeus logo banner (320×180, transparent)
- `app/src/main/res/drawable/tv_banner.png` — Same
- `app/src/main/res/drawable-nodpi/tv_banner.png` — Same
- `app/src/main/res/drawable/app_logo_wordmark.png` — New Zeus wordmark (480×120, transparent)
- `app/src/main/res/drawable/app_logo_mark.png` — Zeus icon only (1080×1080, transparent)
- `app/src/main/java/com/zeus/tv/MainActivity.kt` — Fixed `contentDescription="NuvioTV"` → `"Zeus"`

### Logo Source Files (saved to `C:\ZeusTV\assets\zeus\`)
- `zeus_icon_transparent.png` — Source Z icon (512×512)
- `Cinzel-Bold.ttf` — Font used for "eus" text
- `tv_banner_320x180.png` — Generated TV banner
- `wordmark_480x120.png` — Generated wordmark
- `logo_mark_1080x1080.png` — Generated logo mark

### Logo Spec (from `C:\zeus\src\components\ZeusLogo.tsx`)
- Icon size = canvas height × 0.888
- Font size = iconSize × 0.314
- Text overlap = iconSize × 0.30 (negative margin — "eus" overlaps icon)
- Gradient: #f5a623 (orange) → #8b5cf6 (purple), horizontal

### Originals Backed Up
`C:\ZeusTV\backup_logos\`

---

## [Fork] NuvioTV → ZeusTV — 2026-03-22

### Summary
Complete identity fork of NuvioTV into ZeusTV. No functional code was rewritten.

---

### Phase 1 — Backend Credentials
**Files Created:**
- `local.properties` — Zeus Supabase URL, ANON_KEY, TMDB key, TV login URL, signing config
- `local.dev.properties` — Same credentials for debug builds

**Why:** NuvioTV pointed to the original Supabase project. ZeusTV uses its own backend.

---

### Phase 2 — Branding Assets
**Files Changed:**
- `app/src/main/res/mipmap-hdpi/ic_launcher.png` — Replaced with Zeus icon
- `app/src/main/res/mipmap-mdpi/ic_launcher.png` — Replaced with Zeus icon
- `app/src/main/res/mipmap-xhdpi/ic_launcher.png` — Replaced with Zeus icon
- `app/src/main/res/mipmap-xxhdpi/ic_launcher.png` — Replaced with Zeus icon
- `app/src/main/res/mipmap-xxxhdpi/ic_launcher.png` — Replaced with Zeus icon
- `app/src/main/res/mipmap-*/ic_launcher_round.png` — Replaced with Zeus round icon
- `app/src/main/res/mipmap-*/ic_launcher_foreground.png` — Replaced with Zeus foreground
- `app/src/main/res/drawable/tv_banner.png` — Replaced with Zeus branding
- `app/src/main/res/drawable-nodpi/tv_banner.png` — Replaced with Zeus branding
- `app/src/main/res/mipmap-xhdpi/banner.png` — Replaced with Zeus branding
- `assets/zeustv.png` — Added Zeus logo (was nuviotv.png)

**Why:** All visible assets must reflect ZeusTV brand.

**Note:** TV banner (320×180px) is currently using Zeus_icon_final.jpg as a placeholder.
A proper landscape TV banner should be created for the Play Store listing.

---

### Phase 3 — App Name & Strings
**Files Changed:**
- `app/src/main/res/values/strings.xml` — All "Nuvio" text → "Zeus"
- `app/src/debug/res/values/strings.xml` — "Nuvio Debug" → "Zeus Debug"

**Strings updated:** app_name, auth_notice, supporters text, Trakt sync labels,
layout selection welcome, account sign-in description.

**Not changed:** XML `name` attributes (internal Kotlin resource IDs like
`R.string.auth_notice_nuvio_logged_out`) — changing these would require updating all
referencing Kotlin files and provides no user-visible benefit.

---

### Phase 4 — Build Configuration
**Files Changed:**
- `app/build.gradle.kts`

**Changes made:**
| Item | Before | After |
|------|--------|-------|
| namespace | `com.nuvio.tv` | `com.zeus.tv` |
| applicationId | `com.nuvio.tv` | `com.zeus.tv` |
| Debug applicationId | `com.nuviodebug.com` | `com.zeusdebug.tv` |
| GITHUB_OWNER | `tapframe` | `Iricksta` |
| GITHUB_REPO | `NuvioTV` | `ZeusTV` |
| TV_LOGIN_WEB_BASE_URL default | `app.nuvio.tv/tv-login` | `zeustv.vercel.app/tv-login` |
| Signing env vars | `NUVIO_RELEASE_*` | `ZEUS_RELEASE_*` |
| Keystore fallback path | `../nuviotv.jks` | `zeustv.jks` |
| Baseline profile filter | `com.nuvio.tv.**` | `com.zeus.tv.**` |

- `app/src/main/java/com/zeus/tv/core/di/NetworkModule.kt`
  - User-Agent: `Nuvio/$version` → `Zeus/$version`
  - Placeholder URL: `placeholder.nuvio.tv` → `placeholder.zeustv.app`

---

### Phase 5 — Package Rename
**Scope:** 320 Kotlin source files + 5 test files + 1 baselineprofile file

**Operation:**
1. Copied `com/nuvio/tv/` → `com/zeus/tv/` (main, test, baselineprofile)
2. Replaced all `com.nuvio.tv` references with `com.zeus.tv` in all files
3. Renamed class files:
   - `NuvioApplication.kt` → `ZeusApplication.kt`
   - `NuvioNavHost.kt` → `ZeusNavHost.kt`
   - `NuvioTopBar.kt` → `ZeusTopBar.kt`
   - `NuvioDialog.kt` → `ZeusDialog.kt`
4. Updated class name references throughout all files
5. Updated `AndroidManifest.xml`: `.NuvioApplication` → `.ZeusApplication`
6. Deleted old `com/nuvio/` directory trees

**Verification:** `grep -r "com.nuvio.tv" app/src` returns 0 matches.

---

### Phase 6 — Signing Configuration
**Files:**
- `zeustv.jks` — Copied from `C:\zeus\android\app\zeus.keystore`
- `local.properties` — Added signing credentials

**Keystore details:**
- Key alias: `zeus`
- Store location: `zeustv.jks` (project root)

---

## Rollback Instructions

If anything breaks and you need to revert to NuvioTV:
1. The original NuvioTV codebase is untouched in `C:\NuvioTV` before these changes
  (no git history — recommend creating a git repo before executing future changes)
2. To undo package rename: reverse the sed operations (replace `com.zeus.tv` with `com.nuvio.tv`)
3. To undo config: restore from `local.example.properties`

---

## [Phase 2] Supabase Schema, Auth Whitelist, Sync & Web Update — 2026-03-22

### Phase B — Complete Supabase SQL Schema
**File Created:** `zeustv_schema.sql`
- 11 tables: `family_whitelist`, `profiles`, `addons`, `plugins`, `library_items`, `watch_progress`, `watched_items`, `linked_devices`, `sync_codes`, `avatar_catalog`, `tv_login_sessions`
- 23 RPCs: all sync push/pull, QR login, device linking, whitelist check
- RLS policies on every table
- Derived from `SupabaseModels.kt` field names

**Why:** Zeus Supabase had only `family_whitelist` + `household_sync`. TV app needs 20+ RPCs and 8+ tables.

---

### Phase D — AuthManager.kt Whitelist Checks
**File Changed:** `app/src/main/java/com/zeus/tv/core/auth/AuthManager.kt`
- `signUpWithEmail()`: calls `is_email_whitelisted()` RPC before account creation
- `signInWithEmail()`: queries `family_whitelist` after login, signs out if not active

**Why:** TV app was bypassing whitelist — anyone with the app could create an account.

---

### Phase E — TV Login Exchange Edge Function
**File Created:** `supabase/functions/tv-logins-exchange/index.ts`
- Validates QR code + device_nonce
- Checks whitelist before returning tokens
- Marks session as `exchanged` (one-time use)
- Deployed via: `supabase functions deploy tv-logins-exchange --no-verify-jwt`

---

### Phase F — Web Dashboard Unified Schema
**File Changed:** `C:\zeus\web\index.html`

1. **Data loading** (`handleLoggedIn`): Changed from `household_sync` table to `profiles` + `addons` tables. Maps DB snake_case columns to in-memory camelCase fields.
2. **Data saving** (`saveData`): Changed from `household_sync` upsert to per-profile upsert into `profiles` table + delete/reinsert into `addons` table with position ordering.
3. **TV QR Login** (in `handleLoggedIn`): Changed `?tv=CODE` → `?code=CODE` URL param; replaced `tv_auth_codes` table insert with `approve_tv_login_session()` RPC call.
4. **Connect TV** (`connectTV`): Changed `tv_auth_codes` insert to `approve_tv_login_session()` RPC.
5. **Realtime listener** (`setupRealtime`): Changed from `household_sync` to `profiles` table.
6. **Profile IDs** (`doCreateProfile`): Changed `genId()` to `crypto.randomUUID()` — required for UUID primary key compatibility.

**Why:** Web dashboard was using legacy `household_sync` table. TV app uses `profiles`/`addons` tables. Both must use the same schema for sync to work.
