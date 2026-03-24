# ZeusTV Changelog

## [Fix] Subtitle "None" Not Enforced on New Content — 2026-03-24

**Changes:**
- **Subtitle disabled when preference is "none"** — `tryAutoSelectPreferredSubtitleFromAvailableTracks()` now calls `disableSubtitles()` before returning when the preferred language list is empty (preference = "none"). Previously the function returned without disabling tracks, leaving ExoPlayer's default auto-selector free to pick the first available track (English). This caused subtitles to re-appear on every new movie even though the user had turned them off.

**Files changed:**
- `ui/screens/player/PlayerRuntimeControllerTracks.kt` — added `disableSubtitles()` call in `targets.isEmpty()` branch

---

## [Improvement] Player Cleanup — 2026-03-24

**Changes:**
- **Player job cleanup** — `hideAspectRatioIndicatorJob` is now explicitly cancelled in `releasePlayer()`, consistent with all other player jobs. Previously relied solely on `viewModelScope` auto-cancellation.

---

## [Fix] Subtitle Sync + CW Instant Load — 2026-03-24

**Changes:**
- **CW instant load** — enrichment grace period set to 0ms. Images now appear immediately after app restart using 30-day disk cache. First-ever load still fetches from TMDB (~3-5s) but all subsequent restarts are instant.
- **Subtitle default changed to "none"** — `preferredLanguage` now defaults to `"none"` for all new profiles. Subtitles are off unless the user explicitly enables them.
- **Cross-device subtitle sync** — subtitle preference is now pushed to Supabase as a universal group (`subtitle_prefs`, `device_type = "all"`) whenever changed on TV. Mobile receives this on startup and profile switch.
- **Startup subtitle pull** — `StartupSyncService` now applies the `subtitle_prefs` group (after `player` group) so the cross-device preference takes effect on every TV startup.

**Files changed:**
- `ui/screens/home/HomeViewModel.kt` — `CONTINUE_WATCHING_ENRICHMENT_GRACE_PERIOD_MS` = 0
- `data/local/PlayerSettingsDataStore.kt` — subtitle default `"none"`, universal subtitle_prefs push in init
- `core/sync/StartupSyncService.kt` — applies `subtitle_prefs` group on pull
- `data/local/TmdbEnrichmentCacheDataStore.kt` — new 30-day persistent disk cache for TMDB enrichment

---

## [Fix] CW Card Missing Title + Instant Load After First View — 2026-03-24

**Issues:**
1. CW cards for mobile-played content showed no movie/show title (blank where title should be)
2. Every time the app restarted, CW images/hero took 3-5 seconds to reload

**Root causes:**
1. Supabase only stores watch position (not content name). When TV pulled synced items, title was blank. The enrichment code already fetches metadata (for images) but was discarding the name field.
2. `TmdbMetadataService` had an in-memory-only cache. Every cold start hit the TMDB API again for all CW items (4 parallel API calls per item).

**Fix — 4 files:**
- `HomeUiState.kt` — Added `title: String?` field to `ContinueWatchingItem.InProgress`
- `HomeViewModelContinueWatching.kt` — `enrichInProgressItem()` now populates `title` from `meta.name` / `tmdbData.localizedTitle`
- `ContinueWatchingSection.kt` — Card title now uses enriched title first, falling back to `progress.name`
- `TmdbEnrichmentCacheDataStore.kt` *(new)* + `TmdbMetadataService.kt` — Added persistent DataStore cache (TTL 30 days) following the `StreamLinkCacheDataStore` pattern. Lookup order: memory → disk (~50ms) → TMDB API. First view = slow (API fetch, saved to disk). All subsequent restarts = instant (disk cache hit).

---

## [Fix] CW Card Blank for Mobile-Played Content + Hero Backdrop Slow — 2026-03-24

**Issues:**
1. CW card was black/blank for content watched on mobile or tablet (but showed correctly for TV-played content)
2. Hero backdrop took 3–5 seconds to appear when focusing a CW item

**Root causes:**
1. `ContinueWatchingSection.kt` was reading image URLs from `progress.backdrop` / `progress.poster` which are always null for synced items. The enriched wrapper fields `item.backdrop` / `item.poster` (correctly populated by our TMDB enrichment) were being ignored entirely.
2. `enrichVisibleContinueWatchingItems()` ran TMDB enrichment sequentially in a `forEach` loop — total wait time was the sum of all TMDB calls combined.

**Fix — 2 files:**
- `ContinueWatchingSection.kt` — Extract `enrichedBackdrop`/`enrichedPoster` from `InProgress` item wrapper. Update both `imageModel` branches to prioritise enriched fields over raw `progress.*` fields.
- `HomeViewModelContinueWatching.kt` — Replace sequential `forEach` with `async`/`awaitAll` + `Semaphore(CW_MAX_NEXT_UP_CONCURRENCY)` so all enrichment requests run concurrently (capped at 2 concurrent TMDB calls). Total enrichment time drops from sum-of-all to max-of-any.

---

## [Fix] Continue Watching Backdrop Black for Synced Items — 2026-03-24

**Issue:** CW cards for content watched on mobile showed text (title, description, IMDb, genre) but completely black backdrop/poster image on the TV home screen.

**Root cause (three-layer collapse):**
1. `WatchProgressSyncService.kt` — when pulling watch progress from Supabase, `poster`, `backdrop`, and `logo` are hardcoded to `null`. The Supabase `watch_progress` table stores only playback data, not images.
2. `ContinueWatchingItem.InProgress` data class had no `backdrop`/`poster`/`logo` fields — there was nowhere to store enriched images.
3. `enrichInProgressItem()` enriched text metadata (description, IMDb, genres) but never called TMDB to resolve images. `ModernHomeModels.kt` then read from `item.progress.backdrop` which was always `null`.

**Fix (3 files):**
- `HomeUiState.kt` — Added `backdrop`, `poster`, `logo` fields to `ContinueWatchingItem.InProgress`
- `HomeViewModelContinueWatching.kt` — Updated `enrichInProgressItem()` to resolve TMDB data via `resolveTmdbIdForNextUp()` + `tmdbMetadataService.fetchEnrichment()`, populating images from: progress fields → addon meta → TMDB (in priority order)
- `ModernHomeModels.kt` — Updated hero section and CW card to use enriched `item.backdrop`/`item.poster`/`item.logo` fields with fallback to raw progress fields

---

## [Fix] ProGuard Package Name Crash Fix — 2026-03-24

**Crash:** App crashed every time a profile was selected after the v1.0.1 sync update.

**Root cause:** `proguard-rules.pro` still referenced the old package name `com.nuvio.tv` in 7 keep rules after the fork rename to `com.zeus.tv`. R8 was freely obfuscating domain model, DTO, plugin, server, and Supabase classes. Each new APK build used a different R8 mapping, so the Gson manifest cache from a previous build contained field names that no longer matched → Gson returned `LinkedTreeMap` instead of `CatalogDescriptor` → `ClassCastException` crash in `LayoutSettingsViewModel`.

**Fix:** Updated `app/proguard-rules.pro` — replaced all 7 `com.nuvio.tv` occurrences with `com.zeus.tv` to restore keep rules for:
- `com.zeus.tv.data.remote.api.**`
- `com.zeus.tv.data.remote.dto.**`
- `com.zeus.tv.domain.model.**`
- `com.zeus.tv.core.server.**`
- `com.zeus.tv.core.plugin.**`
- `com.zeus.tv.data.remote.supabase.**`

**After installing the fixed APK:** Run `adb shell pm clear com.zeus.tv` once to clear the corrupted Gson manifest cache from the broken build.

---

## [Feature] Device-Scoped Settings Sync (TV vs Phone) — 2026-03-23

Settings sync now correctly separates TV and phone/tablet settings so they don't overwrite each other.

- **Layout settings** (sidebar, layout type, poster sizes) — syncs per device type. TV layout stays on TV, phone layout stays on phone.
- **Player settings** (decoder, frame rate matching, tunneling) — syncs per device type.
- **Universal settings** (theme, trakt, trailer, anime skip, track preferences) — still sync across all devices as before.

How it works: each device tags its settings as `"tv"` or `"phone"` in Supabase. A TV only pulls TV + universal settings. A phone only pulls phone + universal settings.

---

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
