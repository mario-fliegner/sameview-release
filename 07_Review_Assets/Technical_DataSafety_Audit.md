# Technical Data Safety Audit

**App:** SameView (`com.isardomains.sameview`)
**Repository:** `C:\data\work\privat\git-repos\sameview`
**Audit date:** 2026-07-20
**Auditor:** Claude Code (static repository analysis)
**Scope:** Full-repository static analysis — manifest, Gradle/version catalog, and Kotlin sources — performed to support the Google Play Data Safety declaration for the first public release.

> **Status (2026-10-02): superseded in part**
> The body of this document records the v1.0 state as audited on 2026-07-20 and is kept unchanged as a record. Its findings that `android.permission.INTERNET` is not declared, that no networking library is present, and that the app is "fully offline", as well as its Data Safety mapping and final recommendation, no longer describe the current app. See [Addendum 2026-10-02](#addendum-2026-10-02-optional-deinwackelbild-order-flow) at the end of this document.

> **Purpose of this document**
> This is the authoritative technical reference behind SameView's Google Play Data Safety declaration. It exists so that future submissions, app updates, and Play review questions can be answered by pointing to concrete, verified evidence in the codebase rather than re-deriving the analysis from scratch. It is not a copy of the Play Console questionnaire — it is the reasoning and evidence that justifies each answer chosen there.

---

## Executive Summary

SameView is a fully offline, single-user, on-device camera and photo-comparison app. It has no backend, no account system, and no third-party SDKs that transmit data anywhere. The single most load-bearing fact in this audit is that `AndroidManifest.xml` never declares `android.permission.INTERNET` — this is not a policy choice that could quietly regress, it is an OS-enforced guarantee: without this permission, the app process cannot open a socket or make an HTTP(S) request under any circumstance.

Supporting findings:

- **Dependencies** are limited to AndroidX/Jetpack Compose, CameraX, Media3, Coil, Hilt/Dagger, DataStore, and ExifInterface. No analytics, crash reporting, advertising, or networking library is present anywhere in the dependency graph.
- **Permissions** requested are `CAMERA`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, and `ACCESS_MEDIA_LOCATION`. No storage permission is requested — reference image selection uses the system Photo Picker, which requires no runtime grant.
- **Location access is opt-in and off by default**, gated behind a single Settings toggle ("Recreation guidance") that is `false` until the user explicitly enables it.
- **All user content** — captured photos, GPS coordinates, titles/descriptions, branding assets — is stored exclusively in app-private internal storage (`context.filesDir/sessions/...`) and is explicitly excluded from Android Auto Backup, cloud backup, and device transfer.
- **No background services, WorkManager, or JobScheduler** exist; all processing happens synchronously in the foreground.
- The only outbound-facing surfaces are OS-delegated intents — a browser `ACTION_VIEW` for the website/privacy policy and a mail client `ACTION_SENDTO` for support — neither of which involves the app itself transmitting data.

This audit independently corroborates the repository's own prior internal review (`docs/RELEASE_HARDENING_AUDIT_V2.md`, 2026-07-08), which reached the same conclusion ("kein INTERNET-Permission, kein Tracking... keine Restore-/Import-Angriffsfläche") and flagged the Play Console Data Safety form itself as the one remaining open item this document resolves.

---

## Audit Scope

The following were inspected directly (file paths relative to the repository root):

| Area | Files inspected |
|---|---|
| Manifest | `app/src/main/AndroidManifest.xml`, `app/src/debug/AndroidManifest.xml` |
| Gradle / dependencies | `app/build.gradle.kts`, `gradle/libs.versions.toml`, `build.gradle.kts`, `settings.gradle.kts` |
| Permissions | Full-text search of `app/src/main` for `Manifest.permission.*`, `requestPermission`, `rememberLauncherForActivityResult` |
| Storage | `SessionStorage.kt`, `MediaStoreWriter.kt`, `ShareMediaStoreWriter.kt`, `MediaStoreVideoWriter.kt`, `SessionBackupExporter.kt` |
| Networking | Full-text search for `INTERNET`, `OkHttp`, `Retrofit`, `HttpURLConnection`, `http(s)://` literals |
| Third-party SDKs | Full-text search for Firebase, Crashlytics, AdMob, Bugsnag, Sentry, Mixpanel, Amplitude, AppsFlyer, Facebook, OneSignal, Braze |
| Camera | `CameraScreen.kt`, `CameraViewModel.kt` |
| GPS / Location | `LocationProvider.kt`, `GpsExifWriter.kt`, `CameraViewModel.kt`, `SettingsScreen.kt`, `SettingsModule.kt` |
| EXIF | `GpsExifWriter.kt`, `ReferenceImageMetadataReader.kt`, `SessionStorage.kt` |
| MediaStore / SAF | `MediaStoreWriter.kt`, `ShareMediaStoreWriter.kt`, `MediaStoreVideoWriter.kt`, `CompareLibraryScreen.kt` |
| Sharing | `ShareComparisonScreen.kt`, `ShareComparisonViewModel.kt` |
| Export / Backup | `SessionBackupExporter.kt`, `backup_rules.xml`, `data_extraction_rules.xml` |
| Local databases / preferences | `SettingsRepository.kt` (DataStore Preferences); no Room/SQLite usage found |
| Sensors | `CompassProvider.kt` |
| Accounts / Contacts / Clipboard / Microphone | Full-text search for `AccountManager`, `GoogleSignIn`, `CredentialManager`, `ContactsContract`, `ClipboardManager`, `ClipData`, `RECORD_AUDIO`, `MediaRecorder`, `AudioRecord` — no matches |
| Background work | Full-text search for `WorkManager`, `JobScheduler`, `<service>`, `<receiver>` — no matches |
| Application entry point | `SameViewApplication.kt` |

No code was modified as part of this audit.

---

## Overall Architecture

SameView is architected as a **fully offline application**:

- No backend server exists or is referenced anywhere in the codebase.
- `android.permission.INTERNET` is **not declared** in `AndroidManifest.xml`. This is enforced by the Android OS at the process/UID level — its absence is a hard technical guarantee, not a convention that depends on code discipline elsewhere.
- All persistent user data — photos, GPS metadata, titles, descriptions, branding — lives under app-private internal storage (`context.filesDir`), which is sandboxed by the OS and inaccessible to other apps.
- `sessions/` and `branding/` are explicitly excluded from:
  - Android Auto Backup (`android:fullBackupContent="@xml/backup_rules"`)
  - Cloud backup and device transfer (`android:dataExtractionRules="@xml/data_extraction_rules"`)
- No analytics SDK, crash reporter, or advertising SDK is present in the dependency graph.
- No background execution surfaces (`WorkManager`, `JobScheduler`, `Service`, `BroadcastReceiver`) exist, so there is no path for data processing to occur outside the user's active foreground session.
- The only two data movements that ever cross the app boundary are:
  1. The Android Share Sheet (`ACTION_SEND`), triggered exclusively by explicit user action.
  2. A user-directed Storage Access Framework export (`ACTION_CREATE_DOCUMENT`) writing a ZIP backup to a location the user picks.

---

## Permissions

| Permission | Required | Purpose | Optional | Notes |
|---|---|---|---|---|
| `android.permission.CAMERA` | Yes | Live camera preview and photo capture — the app's core function | No | Backed by `<uses-feature android:name="android.hardware.camera" android:required="true"/>`; the app cannot function without it |
| `android.permission.ACCESS_FINE_LOCATION` | No | Reads device GPS fix to write GPS EXIF into new captures for the "Recreation guidance" feature | Yes | Only requested when the user enables the Settings toggle (`SettingsRepository.recreationGuidance`, default `false`); permission request originates from `SettingsScreen.kt` |
| `android.permission.ACCESS_COARSE_LOCATION` | No | Fallback location provider alongside `ACCESS_FINE_LOCATION` | Yes | Requested together with `ACCESS_FINE_LOCATION` in the same permission launcher call |
| `android.permission.ACCESS_MEDIA_LOCATION` | No | Allows reading GPS EXIF tags already embedded in a user-selected reference photo | Yes | Used defensively in `ReferenceImageMetadataReader.kt` / `SessionStorage.resolveSourceUri`; does not grant access to device location itself |

**Not requested:** `READ_MEDIA_IMAGES`, `READ_EXTERNAL_STORAGE`, `RECORD_AUDIO`, `POST_NOTIFICATIONS`, or any account/contacts permission. Reference image selection uses the system Photo Picker, which requires no runtime permission grant and limits app access to the single file the user selects.

> **Note:** If any permission is added in a future release, this table — and the corresponding Play Console Data Safety answers — must be updated in the same change. Google's review tooling increasingly cross-checks manifest permissions against the Data Safety declaration and can reject a submission for mismatch.

---

## Third-Party Libraries

| Library | Purpose | Sends Data | Notes |
|---|---|---|---|
| AndroidX Core / Lifecycle / Activity-Compose | Core Android/Compose framework support | No | Standard Jetpack foundation libraries |
| Jetpack Compose (BOM, UI, Material3, Material Icons) | UI toolkit | No | No network-capable component |
| CameraX (`camera-camera2`, `camera-lifecycle`, `camera-view`) | Camera preview and capture pipeline | No | Local hardware access only |
| Media3 (`media3-exoplayer`, `media3-ui`) | Local video playback (comparison video preview) | No | Plays locally rendered/exported files only; not configured for streaming/DRM/remote sources |
| Coil (`coil-compose`) | Image loading/caching for local bitmaps and files | No | Used for local file/bitmap display, not remote image loading |
| Hilt / Dagger (`hilt-android`, `hilt-navigation-compose`) | Dependency injection | No | Compile-time DI framework, no runtime network behavior |
| AndroidX DataStore (`datastore-preferences`) | Local key-value settings storage | No | Backed by a local file, never transmitted |
| AndroidX ExifInterface | Read/write EXIF metadata (orientation, GPS, software tag) | No | Operates on local files/URIs only |
| Navigation Compose | In-app screen navigation | No | No network involvement |

**Explicitly absent from the dependency graph** (verified via `gradle/libs.versions.toml`, `app/build.gradle.kts`, and full-text source search): Firebase (any module), Firebase Analytics, Crashlytics, AdMob or any ad network, Retrofit, OkHttp, `HttpURLConnection`/`URLConnection` usage, Bugsnag, Sentry, Mixpanel, Amplitude, AppsFlyer, Facebook SDK, OneSignal, Braze, and any `com.google.android.gms.*` Play Services artifact. No `google-services.json` file exists in the repository.

---

## Data Flow Analysis

### Camera

- **Purpose:** Live preview and photo capture — the app's core function.
- **Data accessed:** Live camera frames via CameraX; captured JPEG output.
- **Storage location:** App-private session directory (`context.filesDir/sessions/<id>/`).
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Photos and videos → Photos.
- **Confidence:** High.

### Location (GPS)

- **Purpose:** Opt-in "Recreation guidance" feature — writes GPS EXIF into a new capture and stores it in `metadata.json` so the user can be guided back to the same physical location for a future comparison.
- **Data accessed:** Latitude, longitude, altitude, accuracy, provider, and fix timestamp via `LocationManager` (`LocationProvider.kt`).
- **Storage location:** Local EXIF tags and `metadata.json` inside the app-private session directory.
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Location → Approximate location / Precise location.
- **Confidence:** High for behavior; Medium on the exact "Collected" checkbox semantics (see [Potential Review Questions](#potential-review-questions)).

### `ACCESS_MEDIA_LOCATION`

- **Purpose:** Allows reading GPS EXIF tags that already exist inside a reference photo the user picked, rather than accessing device location.
- **Data accessed:** Pre-existing GPS EXIF tags of a single user-selected file.
- **Storage location:** N/A — read-only access to the source file's existing metadata.
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Location.
- **Confidence:** Medium.

### EXIF read/write

- **Purpose:** Correct image rotation, date badges in the comparison UI, and (opt-in) GPS EXIF writing.
- **Data accessed:** Orientation, GPS tags, `DateTimeOriginal`, and a `Software` tag ("SameView") written by `GpsExifWriter.kt`; read by `ReferenceImageMetadataReader.kt`.
- **Storage location:** Embedded directly in the JPEG file being processed (session-local or MediaStore entry).
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Covered under Photos and videos / Location — no separate line item.
- **Confidence:** High.

### Photo Picker (reference image selection)

- **Purpose:** Lets the user choose the "reference" photo to compare against.
- **Data accessed:** A single image file selected via the system Photo Picker.
- **Storage location:** Referenced by content URI; copied into the session directory as `reference-source-original.*`.
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Photos and videos.
- **Confidence:** High — no `READ_MEDIA_IMAGES` permission is requested, confirming the app never gains broad gallery access.

### MediaStore image export (Share / Save)

- **Purpose:** Saves or shares a finished comparison image outside the app.
- **Data accessed:** Rendered JPEG comparison image. `ShareMediaStoreWriter.kt` writes **no** EXIF/GPS/XMP/IPTC metadata at all; `MediaStoreWriter.kt` writes a `Software` tag and optional GPS.
- **Storage location:** `MediaStore.Images` under `Pictures/SameView`.
- **Leaves device?** Only if the user actively selects a destination app from the Android Share Sheet — that transmission is performed by the receiving app, not SameView.
- **Shared?** No automatic sharing. Manual share is via `ACTION_SEND` (`type = "image/jpeg"`, `FLAG_GRANT_READ_URI_PERMISSION`) in `ShareComparisonScreen.kt`.
- **Play category:** Photos and videos.
- **Confidence:** High.

### Video export

- **Purpose:** "Create video" comparison export feature.
- **Data accessed:** Session images rendered into a silent MP4 (no audio track; no `MediaRecorder`/`AudioRecord`/`RECORD_AUDIO` usage anywhere in the `video/` package).
- **Storage location:** `MediaStore` video collection via `MediaStoreVideoWriter.kt`.
- **Leaves device?** Only via explicit user Share action.
- **Shared?** No, unless the user shares.
- **Play category:** Photos and videos → Videos.
- **Confidence:** High.

### Session storage

- **Purpose:** Core data model of the app — every comparison "session" the user creates.
- **Data accessed:** `capture.jpg`, `capture-original.jpg`, `reference.jpg`, `reference-original.jpg`, `reference-source-original.*`, `branding-handle.png`, and `metadata.json` (GPS, dates, user-entered title/description, favorite flag, optional location display name/city/country).
- **Storage location:** `context.filesDir/sessions/<session-id>/` — app-private, not shared storage.
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** Photos and videos + Location + App activity → Other user-generated content (title/description).
- **Confidence:** High.

### DataStore (app preferences)

- **Purpose:** Persists UI/feature preferences across launches — grid type, keep-screen-on, overlay-reset behavior, auto-open-compare, recreation-guidance toggle, live-direction-arrow, branding-enabled, library filter/sort order, strip-metadata toggle.
- **Data accessed:** Booleans/enums only — no personal data, no identifiers.
- **Storage location:** Local DataStore Preferences file (`SettingsRepository.kt`).
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** None — pure app-preference values are not a disclosable personal data type.
- **Confidence:** High.

### ZIP backup (Storage Access Framework export)

- **Purpose:** User-initiated manual backup of one or more comparison sessions.
- **Data accessed:** Full session bundles (photos + `metadata.json`), copied byte-for-byte by `SessionBackupExporter.kt`.
- **Storage location:** Written to a destination the user selects via `ActivityResultContracts.CreateDocument("application/zip")` (`CompareLibraryScreen.kt`).
- **Leaves device?** Only to a location the user themselves chooses (which may be a cloud-synced folder — that is the user's own OS-level choice, not an action initiated by SameView).
- **Shared?** No. Export-only — no import/restore path exists anywhere in the codebase (confirmed by full-text search for `ACTION_OPEN_DOCUMENT` and import-related identifiers).
- **Play category:** Files and docs.
- **Confidence:** High.

### Auto Backup / Cloud Backup / Device Transfer exclusions

- **Purpose:** Prevents user photos and branding assets from silently syncing via Android's built-in backup infrastructure.
- **Data accessed:** N/A — this is an exclusion rule, not a collection point.
- **Storage location:** Declared in `backup_rules.xml` (`android:fullBackupContent`) and `data_extraction_rules.xml` (`android:dataExtractionRules`), excluding `sessions/` and `branding/` from both cloud-backup and device-transfer domains.
- **Leaves device?** No — that is the explicit purpose of the exclusion.
- **Shared?** No.
- **Play category:** None directly, but strengthens the "not shared" answer for Photos and Location.
- **Confidence:** High.

### Compass (rotation-vector sensor)

- **Purpose:** Drives a live directional arrow overlay to help the user physically orient toward the original photo's direction.
- **Data accessed:** `TYPE_ROTATION_VECTOR` sensor readings, converted to an azimuth float (`CompassProvider.kt`).
- **Storage location:** In-memory only; never persisted.
- **Leaves device?** No.
- **Shared?** No.
- **Play category:** None — Play's Data Safety taxonomy has no motion/orientation sensor category, and no `ACTIVITY_RECOGNITION`/body-sensor permission is used.
- **Confidence:** High.

### Support email

- **Purpose:** User-initiated support contact.
- **Data accessed:** None by the app itself — `ACTION_SENDTO` with `mailto:support@sameview.app` opens the user's own email client with a pre-filled subject (`AboutScreen.kt`).
- **Storage location:** N/A.
- **Leaves device?** Only if the user sends the email via their own mail app.
- **Shared?** No app-side sharing — SameView never reads or stores the email content.
- **Play category:** None.
- **Confidence:** High.

### Privacy Policy / Website link

- **Purpose:** Marketing site and required privacy policy link.
- **Data accessed:** None by SameView — `ACTION_VIEW` opens `https://sameview.app` or `https://sameview.app/en/privacy` in the user's own browser (`AboutScreen.kt`).
- **Storage location:** N/A.
- **Leaves device?** The browser navigation happens outside the app's process; SameView itself makes no network call.
- **Shared?** N/A.
- **Play category:** None for the app itself. The privacy policy URL is a required, separate field in the Play Console Data Safety section (not tied to a data type).
- **Confidence:** High — verify the live URL resolves without a login wall before submission.

### Third-party SDKs

- **Purpose:** N/A — no data-transmitting third-party SDK exists in the dependency graph.
- **Data accessed:** None.
- **Storage location:** N/A.
- **Leaves device?** N/A.
- **Shared?** N/A.
- **Play category:** None.
- **Confidence:** High.

### Accounts / Contacts / Clipboard / Microphone

- **Purpose:** N/A — none of these surfaces exist in the app.
- **Data accessed:** None. No `AccountManager`, `GoogleSignIn`, `CredentialManager`, `ContactsContract`, `ClipboardManager`/`ClipData`, `RECORD_AUDIO`, `MediaRecorder`, or `AudioRecord` usage was found anywhere in `app/src/main`.
- **Storage location:** N/A.
- **Leaves device?** N/A.
- **Shared?** N/A.
- **Play category:** None.
- **Confidence:** High.

### Background work

- **Purpose:** N/A — all processing (capture, export, backup) happens synchronously while the app is in the foreground.
- **Data accessed:** None. No `WorkManager` dependency, no `JobScheduler`, and no declared `<service>` or `<receiver>` in the manifest.
- **Storage location:** N/A.
- **Leaves device?** N/A.
- **Shared?** N/A.
- **Play category:** None.
- **Confidence:** High.

---

## Google Play Data Safety Mapping

| Play Category | Collected | Shared | Purpose | Optional | Confidence |
|---|---|---|---|---|---|
| Location → Approximate location | Yes | No | App functionality | Optional | High |
| Location → Precise location | Yes | No | App functionality | Optional | High |
| Photos and videos → Photos | Yes | No | App functionality | Required (core feature) | High |
| Photos and videos → Videos | Yes | No | App functionality | Optional (video export feature) | High |
| App activity → Other user-generated content (title/description/favorite) | Yes | No | App functionality | Optional | High |
| Files and docs | Yes | No | App functionality | Optional (backup export feature) | High |
| Personal info | No | — | — | — | High |
| Financial info | No | — | — | — | High |
| Health and fitness | No | — | — | — | High |
| Messages | No | — | — | — | High |
| Audio files | No | — | — | — | High |
| Contacts | No | — | — | — | High |
| Calendar | No | — | — | — | High |
| App info and performance | No | — | — | — | High |
| Device or other IDs | No | — | — | — | High |
| Web browsing | No | — | — | — | High |

---

## Security & Privacy

- **On-device processing only.** Every data flow analyzed above — camera capture, location, EXIF read/write, session storage, video rendering — executes entirely within the app's own process and app-private storage.
- **No network communication.** The absence of `android.permission.INTERNET` is an OS-enforced technical guarantee, not a code-review convention. It applies uniformly to every component in the app, including any third-party library, since none of the declared dependencies request network access either.
- **No background upload.** No `WorkManager`, `JobScheduler`, `Service`, or `BroadcastReceiver` exists, so there is no mechanism for data to be processed or transmitted outside an active foreground session.
- **Backup exclusions are explicit and verified.** `sessions/` and `branding/` are excluded from Android Auto Backup, cloud backup, and device transfer via `backup_rules.xml` and `data_extraction_rules.xml` — user photos never leave the device even through Google's own backup infrastructure.
- **Exports are manual and user-directed only.** The two mechanisms that move data outside the app sandbox — the Android Share Sheet and the SAF ZIP backup — both require an explicit, single user action per invocation. No import/restore path exists, eliminating a class of attack surface around malformed or spoofed backup files.

---

## Potential Review Questions

### 1. Is on-device-only Location/Photos access still "Collected"?

Google's Data Safety guidance on what counts as "collected" has shifted over time and is not unambiguous for data that is accessed via a dangerous permission (`ACCESS_FINE_LOCATION`, `CAMERA`) but never transmitted off the device. The lower-risk, defensible position taken in this audit is to declare Location and Photos as **"Collected: Yes, Shared: No"** rather than omitting them — omitting a data type backed by a dangerous permission is the more common cause of Data Safety policy strikes, not over-declaring a type that is honestly marked as not shared. This determination should be re-checked against the live Play Console form text at submission time, since Google periodically revises the exact wording.

### 2. Does the privacy policy URL meet Play's requirements?

`https://sameview.app/en/privacy` must resolve publicly, without a login wall, and its content must match the practices documented in this audit (opt-in location, local-only storage, manual export, no third-party sharing). This was flagged as an open blocker in the repository's own prior internal audit (`docs/RELEASE_HARDENING_AUDIT_V2.md`) and must be re-verified as live before each submission.

### 3. What happens if a future release adds a new permission or SDK?

Any new permission in `AndroidManifest.xml` or any new dependency in `gradle/libs.versions.toml` must be reflected in this document and in the corresponding Play Console Data Safety answers in the same release. Google's review tooling cross-checks declared permissions against the Data Safety declaration, and a mismatch is a plausible cause of rejection or post-release enforcement. This audit should be re-run whenever the manifest or dependency graph changes.

### 4. Does the manual export/backup behavior introduce any risk?

The ZIP backup feature (`SessionBackupExporter.kt`) is export-only — there is no corresponding import/restore feature anywhere in the codebase. Because the destination is chosen entirely by the user through the system's Storage Access Framework picker, the app never decides where data goes; it also cannot be tricked into writing files to a location it doesn't have explicit user-granted access to. This significantly narrows the scope of any question a reviewer might raise about the backup/export feature, since there is no attacker-controlled or automatic component to it.

### 5. Does the app's Share Sheet usage constitute "sharing" under Play's definition?

No. Play's Data Safety definition of "shared" refers to a transmission the developer's app itself performs to a third party, not a manual OS-level action initiated entirely by the user (such as picking a destination app from `Intent.createChooser`). SameView never selects a recipient or destination on the user's behalf.

---

## Final Recommendation

Based on the verified technical evidence in this audit, the Google Play Data Safety questionnaire should be completed as follows:

- **Does your app collect or share any of the required user data types?** → **Yes**, due to Location and Photos/Videos access under dangerous permissions.
- **Location (Approximate and Precise)** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Optional**.
- **Photos and videos → Photos** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Required** (core feature).
- **Photos and videos → Videos** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Optional**.
- **App activity → Other user-generated content** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Optional**.
- **Files and docs** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Optional** (backup export feature).
- **All other categories** (Personal info, Financial info, Health and fitness, Messages, Audio files, Contacts, Calendar, App info and performance, Device or other IDs, Web browsing) → **Not collected**.
- **Data encrypted in transit** → Not applicable / no transmission occurs, since `android.permission.INTERNET` is never declared.
- **Data deletion mechanism** → **Yes** — all data is stored locally; users delete it by deleting sessions in-app or by uninstalling the app.
- **Privacy policy URL** → `https://sameview.app/en/privacy`, pending a final live-reachability check before submission.

These answers are technically supported by direct evidence in the codebase: the absence of `android.permission.INTERNET` in the manifest, the absence of any data-transmitting third-party SDK in the dependency graph, the app-private storage location of all user content (`context.filesDir/sessions/...`), the explicit backup/cloud/device-transfer exclusions for that storage, the opt-in and default-off state of the location feature, and the fully manual, user-initiated nature of every mechanism (Share Sheet, ZIP export) that can move data outside the app sandbox. No aspect of this declaration depends on a promise about future behavior — every claim here traces to a specific, currently-verified file or code path in the audited repository.

---

## Addendum 2026-10-02: Optional DeinWackelbild Order Flow

**Addendum date:** 2026-10-02
**Author:** Claude Code
**Scope:** Delta only. This addendum records what changed for the Data Safety declaration after the app gained the optional DeinWackelbild order. It is **not** a full re-audit: it is based on the established product facts for this feature, plus a check of `app/src/main/AndroidManifest.xml`, `app/build.gradle.kts`, and `gradle/libs.versions.toml`. The Kotlin sources were not re-inspected. Everything in the body above that is not listed here is left as audited on 2026-07-20.

### Superseded statements

| Statement in the body | Current state |
|---|---|
| `android.permission.INTERNET` is not declared (Executive Summary, Overall Architecture, Security & Privacy, Final Recommendation) | `INTERNET` is declared, for the optional DeinWackelbild order. |
| "Fully offline" application | Normal use remains local/offline. The optional order transmits data after an explicit user action. |
| No networking library in the dependency graph; OkHttp explicitly absent | OkHttp and AndroidX Browser are dependencies, used for the optional order. |
| The only data movements across the app boundary are the Share Sheet and the SAF ZIP export | A third one exists: the optional DeinWackelbild order, performed by the app itself. |
| Data Safety mapping: "Collected: Yes" for on-device data | Replaced by the revised mapping below. |
| Encryption in transit: not applicable | Yes (HTTPS). |

### Permissions

| Permission | Required | Purpose | Optional | Notes |
|---|---|---|---|---|
| `android.permission.INTERNET` | No | Transfer for the optional DeinWackelbild order | Yes (feature is optional) | Declared in `AndroidManifest.xml`; normal permission, no runtime prompt |

`ACCESS_NETWORK_STATE` is present in the final release manifest transitively through AndroidX Media3. SameView-owned code does not use it, and it does not itself permit data transmission.

### Third-Party Libraries

| Library | Purpose | Sends Data | Notes |
|---|---|---|---|
| OkHttp (`okhttp`) | HTTPS transfer to DeinWackelbild | Yes, only in the optional order | No other use established |
| AndroidX Browser (`browser`) | Opens the external checkout/configurator in a Chrome Custom Tab | No (the browser performs the navigation) | — |

No analytics, telemetry, tracking, advertising, or crash-reporting SDK is introduced.

### Data Flow: Optional DeinWackelbild order

- **Purpose:** Optional order of a physical lenticular print (Wackelbild).
- **Trigger:** Explicit user action only. Immediately before the transfer, the app discloses that the two images go to DeinWackelbild.de and that ordering is completed there. No automatic upload, no background upload.
- **Data accessed:** Exactly two prepared JPEGs (reference + capture). They are derived/prepared copies, not the stored originals, and contain no EXIF/GPS metadata. If the date option is enabled, the date text is rendered into the image pixels. Technical handoff values include a partner identifier, format, orientation, and direction. No SameView session ID and no external reference is sent.
- **Storage location:** N/A on the SameView side — SameView has no server and does not receive the transferred images back.
- **Leaves device?** Yes, over HTTPS, to O.Schulze / M.Monka GbR (DeinWackelbild), Wolfener Str. 32-34, 12681 Berlin, Germany.
- **After the transfer:** The checkout/configurator opens externally in a Chrome Custom Tab/browser. The print order is completed with DeinWackelbild under its own terms and is separate from the SameView purchase on Google Play.
- **Retention at the recipient:** If no order is completed, DeinWackelbild retains the transferred images for 24 hours and then deletes them. Retention for completed orders is not established here.
- **Commercial relationship:** SameView receives a 10% commission on orders attributable to this handoff.
- **Play category:** Photos and videos → Photos.
- **Confidence:** Based on established product facts, not on source inspection in this addendum.

The partner identifier value is a secret and must never be recorded in this repository.

### Revised Google Play Data Safety Mapping

Google's Data safety guidance defines "collect" as transmitting data from the app off a user's device; data that is only processed locally on the device does not need to be disclosed. This resolves [Potential Review Question 1](#1-is-on-device-only-locationphotos-access-still-collected): on-device-only data is not declared. This also matches the v1.0 submission record (`05_Releases/2026-07-24_PlayStoreSubmission.docx`), in which no data types were selected.

| Play Category | Collected | Shared | Purpose | Optional | Basis |
|---|---|---|---|---|---|
| Photos and videos → Photos | Yes | No | App functionality | Optional | Two prepared JPEGs transmitted in the optional order; not processed ephemerally |
| Photos and videos → Videos | No | — | — | — | Processed on device only |
| Location (Approximate / Precise) | No | — | — | — | Processed on device only; not part of the transfer |
| App activity → Other user-generated content | No | — | — | — | Processed on device only |
| Files and docs | No | — | — | — | Processed on device only |
| All other categories | No | — | — | — | Unchanged |

**Shared: No** for Photos rests on the Google Play exception for a transfer to a third party "based on a specific user-initiated action, where the user reasonably expects the data to be shared". The "service provider" exception is not relied on.

### Revised Final Recommendation

- **Does your app collect or share any of the required user data types?** → **Yes**.
- **Photos and videos → Photos** → Collected: **Yes** · Shared: **No** · Purpose: **App functionality** · **Optional** · not processed ephemerally.
- **All other categories** → **Not collected** (on-device only, or not accessed).
- **Data encrypted in transit** → **Yes** (HTTPS).
- **Data deletion** → to be answered deliberately in the live form. The 24-hour deletion by DeinWackelbild covers transfers without a completed order only and is not used here as the basis for a deletion answer.
- **Privacy policy URL** → unchanged; the live page must describe the optional order.

The repository reference for these answers is `04_PlayConsole/DataSafety.txt`.

### Open Questions

- **Commission and the sharing exception.** The official guidance consulted does not address a commercial relationship in connection with the user-initiated-transfer exception. To be checked when the live Play Console form is completed. No conclusion is drawn here.
- **Deletion question.** Retention for completed orders is not established.
- **External checkout.** The guidance consulted addresses webviews, not Chrome Custom Tabs.
- **Live form.** The current live answers have not been inspected; this addendum changes nothing in the Play Console.
- **Outside Data Safety, not decided here:** SameView's role under GDPR for the transfer, the legal basis for the transfer, and what order or attribution information SameView receives in connection with the commission.

Google sources consulted on 2026-10-02: Play Console Help "Provide information for Google Play's Data safety section" (answer/10787469) and the Play "User Data" policy (answer/10144311).
