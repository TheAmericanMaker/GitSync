# Architecture Map

**Project:** GitSync (theamericanmaker/gitsync, branch `claude/codecarto-gitsync-analysis-goJP2`, source SHA `e6afa871`)
**Phase:** architecture (1/7 of `pipeline-full-with-deep-audit.yaml`)
**Generated:** 2026-05-06

## System Intent

GitSync is a cross-platform mobile git client (Android 5+ / iOS 13+) whose central job is to **keep a chosen local directory continuously synced with a git remote with minimal user intervention**. It packages a Flutter UI on top of a Rust core (libgit2 via `git2-rs`, bridged with `flutter_rust_bridge`) and adds a substantial mobile-platform integration layer — Android quick-tile services, accessibility-event-driven sync, home-screen widgets, an iOS BGTaskScheduler pool, OAuth flows for GitHub/GitLab/Gitea/Codeberg, AI features (chat, autocomplete, agent), and a per-repo "container" model. The target audience is mobile users who treat their phone as a working git surface — note-takers (Obsidian-vault syncing is an explicit acknowledged use case, see the `obisidian_git_found` dialog), single-developer mobile workflows, and people who want repository operations from a quick-settings tile or home-widget rather than a desktop. *(observed fact — README.md, `pubspec.yaml`, AndroidManifest.xml, `lib/ui/dialog/obisidian_git_found.dart`)*

## Layer Map

### Package Inventory

| Package / Module | Role | Public Entrypoints | Key Dependencies | Runtime Surface |
|---|---|---|---|---|
| `rust/src/api/` (`git_manager.rs` 165KB) | core semantics | `frb_generated.rs`-exposed FFI entrypoints (clone/fetch/pull/push/commit/diff/branch/credentials/etc.) | `git2` 0.20.4 (vendored libgit2+openssl), `ssh-key`/`osshkeys` (key parsing/encryption), `libssh2-sys`, `nix`, `regex`, `tokio` (rt+time only), `uuid` | Rust cdylib/staticlib loaded by Flutter at runtime |
| `lib/src/rust/` (generated, excluded from analysis) | protocol or normalization | `flutter_rust_bridge` Dart bindings | flutter_rust_bridge 2.12.0 | Auto-regenerated from `rust/src/api/` by `flutter_rust_bridge_codegen` |
| `lib/api/manager/` (`git_manager.dart` 53KB) | integration adapter | `GitManager` static API (Dart-side wrapper over Rust core) | lib/src/rust, lib/api/helper, settings_manager | Called by gitsync_service, UI dialogs, AI tool layer |
| `lib/api/manager/auth/` (`github_manager.dart` 70KB, `gitea_manager.dart` 49KB, `gitlab_manager.dart` 47KB, `git_provider_manager.dart` 9KB, plus `codeberg`/`github_app` thin wrappers) | integration adapter | `GitProviderManager` (multiplexer) and per-provider clients | `http`, `oauth2_client`, `flutter_secure_storage`, `lib/type/*` | REST clients for issues/PRs/releases/workflows over HTTPS |
| `lib/api/` (AI subsystem: 11 files, ~150KB total) | integration adapter | `AiChatService`, `AiCompletionService`, `AiStreamClient`, `AiToolExecutor`, `AiTools*` registries | `http`, lib/api/manager/git_manager, lib/api/manager/auth, secure-storage | LLM/SSE callouts; tool dispatch back into git/provider/file managers |
| `lib/api/manager/storage.dart` + `settings_manager.dart` + `repo_manager.dart` + `premium_manager.dart` | persistence or state | `Storage` base class, `SettingsManager`, `RepoManager`, `PremiumManager` | `shared_preferences`, `flutter_secure_storage`, `mmap2`, `path_provider` | Per-app and per-repo settings; OAuth tokens; encrypted backups; entitlement state |
| `lib/api/helper.dart` (29KB), `logger.dart` (12KB), `accessibility_service_helper.dart`, `colour_provider.dart`, `issue_template_parser.dart` | integration adapter | Cross-cutting utilities | `flutter_local_notifications`, `mixin_logger`, network helpers | Logger sinks, error-toast helpers, color/theme state, debounce primitive |
| `lib/gitsync_service.dart` (17.5KB) | core semantics | `GitsyncService` (event constants, `debouncedSync`, `merge`, `accessibilityEvent`) | flutter_background_service, flutter_local_notifications, home_widget, workmanager, lib/api/manager/git_manager | The sync orchestration loop — invoked by all sync triggers (open/close, schedule, tile, widget, intent) |
| `lib/providers/riverpod_providers.dart` (21KB) | persistence or state (reactive) | Single root file of Riverpod providers | `flutter_riverpod` | Reactive state graph consumed by UI |
| `lib/type/` (11 files) | core semantics (value types) | `Issue`, `IssueDetail`, `PullRequest`, `PrDetail`, `Release`, `Tag`, `ActionRun`, `AiChat`, `IssueTemplate`, `GitProvider`, `ShowcaseFeature` | none | Wire/value types shared by provider clients and UI |
| `lib/constant/` (8 files; `strings.dart` 11KB, `values.dart` 4KB, `langDiff/langLog`, `secrets.dart.template`) | core semantics | Constants & static strings | none | Constants and the OAuth-secrets template (real `secrets.dart` injected at CI build time, not in repo) |
| `lib/l10n/` (9 ARB sources + 9 generated dart locales + 134.9KB root delegate) | UI or rendering (localization) | `AppLocalizations` delegate | flutter_localizations, intl | i18n table consumed by UI |
| `lib/ui/component/` (23 files; ~225KB total) | UI or rendering | Reusable widgets (auth forms, branch selector, diff view, sync toggles, etc.) | flutter, lib/api, lib/type, riverpod | Composed by dialogs and pages |
| `lib/ui/dialog/` (56 files; `manual_sync.dart` 84KB, `merge_conflict.dart` 61KB, `auth.dart` 22KB, `diff_view.dart` 23KB) | UI or rendering | Dialogs for confirms, auth, manual sync, merge conflict resolution, repo create, github issue reporter, etc. | flutter, lib/api, lib/ui/component | Modal flows |
| `lib/ui/page/` (20 files; `onboarding_setup.dart` 159KB, `global_settings_main.dart` 54KB, `pr_detail_page.dart` 52KB, `file_explorer.dart` 48KB, `ai_features_page.dart` 80KB) | UI or rendering | Top-level pages reachable from main app shell | flutter, lib/api, lib/api/manager/auth, lib/type | Full-screen flows |
| `lib/main.dart` (290KB) | product shell | `main()`, app shell, navigation, top-level state, deep-link handling, background-callback registration | everything below | Single-file Flutter app entry; binds platform-channel handlers from `flutter_background_service`, `home_widget`, `quick_actions`, `workmanager`, `flutter_local_notifications`, `flutter_web_auth_2` |
| `android/app/src/main/kotlin/com/viscouspot/gitsync/` (10 Kotlin files) | product shell | `MainActivity`, `GitSyncAccessibilityService`, `GitSyncService`, `GitTileSyncService`, `GitTileManualSyncService`, `widget/{Force,Manual}SyncWidget{,Receiver}` | Android SDK, flutter_background_service plugin embedding | Android-side accessibility service, foreground services, quick-tile services, App Widget receivers |
| `ios/Runner/` (4 Swift + Info.plist + entitlements) | product shell | `AppDelegate`, `BackgroundIntent`, `SyncNowIntent`, `Runner-Bridging-Header.h` | iOS SDK | iOS app delegate, App Intents, BGTaskScheduler hooks |
| `.maestro/` (Rust crate + YAML flow files) | tooling (excluded from product) | Rust harness driving Maestro UI flows; screenshot generator | Rust crate | UI test runner; out-of-band relative to the app at runtime |

### Dependency Direction

The dependency graph is a clean stack with **no cycles between layers** at the package level. Top to bottom:

```
                   [lib/main.dart]  (Flutter app shell)
                          |
              +-----------+-----------+
              v           v           v
     [lib/ui/page]  [lib/ui/dialog]  [lib/ui/component]
              \           |           /
               \          v          /
                [lib/providers/riverpod_providers]
                          |
              +-----------+-----------+
              v                       v
        [lib/gitsync_service]    [lib/api/* (AI, helpers,
              |                      logger, color, accessibility)]
              v
     [lib/api/manager]  <-----  [lib/api/manager/auth]
              |                       |
              v                       v
     [lib/src/rust (FRB Dart)]   [http / oauth2_client /
              |                    flutter_secure_storage /
              v                    home_widget / workmanager /
     [rust/src/api/git_manager.rs]  flutter_background_service]
              |
              v
     [git2 / libgit2 (vendored) / openssl (vendored) / ssh-key / libssh2]
```

Key direction observations *(strong inference, traced via imports in `lib/global.dart`, `lib/gitsync_service.dart`, `lib/api/manager/repo_manager.dart`, `lib/api/manager/auth/git_provider_manager.dart`)*:

- **Rust core (`rust/src/api/git_manager.rs`) is the lowest stable layer.** Nothing inside the repo depends on the Rust crate other than the FRB-generated bindings and the build pipeline. It pulls in vendored libgit2/openssl as third-party shaping forces.
- **Provider clients (`lib/api/manager/auth/*`) are siblings of the Rust git core**, not stacked above it. They use `http`/`oauth2_client` and never call into the Rust layer; conversely the Rust layer never knows about GitHub/GitLab/Gitea — it speaks HTTPS to whatever git-protocol URL the user provided.
- **`lib/gitsync_service.dart` is the runtime fulcrum.** It is the single orchestration loop that combines the git core, the settings/repo persistence, and the platform-channel surface (widgets, notifications, workmanager, background-service). UI flows ultimately invoke it for every sync; platform triggers (intents, tiles, schedules, accessibility events) ultimately route to it.
- **`lib/global.dart` introduces top-level singletons** (`repoManager`, `uiSettingsManager`, `gitSyncService`, `premiumManager`, `colours`, `aiChatService`, `t` for localizations) plus three `ValueNotifier`/callback hooks (`aiKeyConfigured`, `aiFeaturesEnabled`, `switchToAiTab`). This is a **portability hazard**: the global module is imported throughout the codebase as the source of cross-cutting state, conflating DI/IoC with module-load-order assumptions. *(portability hazard — `lib/global.dart` lines 1-23)*
- **Forks of upstream packages override critical pieces.** Three Dart packages are pinned to ViscousPot forks via `pubspec.yaml` `dependency_overrides`: `flutter_background_service{,_platform_interface,_android,_ios}` (renamed `flutter_background_service_with_intents`), `mmap2_flutter` (custom build), `ios_document_picker` (custom build); plus `workmanager` is pinned to a `main` ref of fluttercommunity's repo. **A port that uses upstream packages will see different intent-broadcast behavior, mmap semantics, and document-picker UX.** *(portability hazard — `pubspec.yaml` lines 32-65, 73-89)*
- **No package-level cycles, but `lib/main.dart` is a structural cycle risk.** At 290KB it imports from every layer below it and is the registration site for every plugin's background callback. Any port must extract the background-callback registrations and the navigation graph before they can be reused. *(strong inference — file size + role)*

## Public Surfaces

### Mobile UI surfaces (user-facing screens/workflows)

| Surface | Owner | Notes |
|---|---|---|
| Onboarding wizard | `lib/ui/page/onboarding_setup.dart` (159KB) | Single-file wizard covering welcome, permissions, accessibility-service enablement, all-files-permission disclosure, author details, auth, repo cloning. Maestro flows under `.maestro/onboarding/` exercise positive and negative paths. *(observed fact)* |
| App home / per-repo dashboard | `lib/main.dart` + `lib/ui/page/sync_settings_main.dart` | Container picker, sync now, recent commits view, AI tab. *(observed fact)* |
| AI features hub | `lib/ui/page/ai_features_page.dart` (80KB) | Chat, autocomplete (wand), agent. Gated by `aiFeaturesEnabled` global. *(observed fact)* |
| File explorer + code editor + image viewer | `lib/ui/page/{file_explorer,code_editor,image_viewer}.dart` | In-app browse/edit. `re_editor` + `re_highlight` for syntax highlighting. *(observed fact)* |
| Diff view | `lib/ui/dialog/diff_view.dart` + `component/diff_file.dart` | File, line, and commit diffs. *(observed fact)* |
| Manual sync orchestrator | `lib/ui/dialog/manual_sync.dart` (84KB) | Large dialog presenting commit/stage/diff/conflict pre-sync triage. *(observed fact)* |
| Merge conflict resolver | `lib/ui/dialog/merge_conflict.dart` (61KB) | In-app conflict resolution flow. *(observed fact)* |
| GitHub Actions runs viewer | `lib/ui/page/actions_page.dart` | OAuth-only feature. *(observed fact)* |
| Issues / PR / Releases / Tags pages | `lib/ui/page/{issues,pull_requests,releases,tags,issue_detail,pr_detail,create_issue,create_pr}_page.dart` | OAuth-only. *(observed fact)* |
| Auth flows | `lib/ui/dialog/auth.dart` + `https_auth_form.dart` / `ssh_auth_form.dart` + `import_priv_key.dart` | HTTPS, SSH, OAuth (GitHub/GitLab/Gitea/Codeberg). *(observed fact)* |
| Settings (global + per-repo + sync) | `lib/ui/page/global_settings_main.dart` (54KB), `settings_main.dart` (35KB), `sync_settings_main.dart` (2.5KB) | Plus per-feature setting components (`auto_sync_settings`, `quick_sync_settings`, `scheduled_sync_settings`). *(observed fact)* |
| 50+ confirm/info dialogs | `lib/ui/dialog/confirm_*` | Each git-mutating operation has a dedicated confirm dialog. *(observed fact)* |

### OS-integration surfaces (platform shell)

| Surface | Owner | Direction | Notes |
|---|---|---|---|
| Android `MainActivity` | `android/.../MainActivity.kt` (1.4KB) | inbound | Hosts Flutter engine; `singleTop` + `gitsync://` deep-link via `flutter_web_auth_2.CallbackActivity`. *(observed fact — AndroidManifest.xml lines 23-46)* |
| Android accessibility service | `GitSyncAccessibilityService.kt` (1.4KB) + `AccessibilityServiceHelper.kt` (9.4KB) | inbound | Listens for window-focus/package-change events; triggers sync on foreground-app open or close per repo's `setman_syncOnAppOpened` / `setman_syncOnAppClosed`. Routed via `lib/gitsync_service.dart:accessibilityEvent()`. *(observed fact)* |
| Android foreground sync service | `GitSyncService.kt` (1.1KB) | inbound | Receives `INTENT_SYNC` action; an `<intent-filter>` with `<action android:name="INTENT_SYNC"/>` makes this an exported, callable service for external automation (Tasker, Macrodroid). *(observed fact — AndroidManifest.xml line 71-77)* |
| Quick Settings tiles | `GitTileSyncService.kt`, `GitTileManualSyncService.kt` | inbound | Two QS tiles: "Sync" (force) and "Manual Sync" (debounced). Each is a TileService bound by `BIND_QUICK_SETTINGS_TILE`. *(observed fact)* |
| Home-screen widgets (Android) | `widget/ForceSyncWidget*.kt`, `widget/ManualSyncWidget*.kt` (~10KB total) | inbound | Two AppWidget receivers, each updated via `home_widget` package; status round-trips through `forceSyncWidget_status` SharedPreferences key. *(observed fact — `lib/gitsync_service.dart:_updateForceSyncWidget`)* |
| iOS App Delegate / Intents | `ios/Runner/AppDelegate.swift` (12.9KB), `BackgroundIntent.swift`, `SyncNowIntent.swift` | inbound | App Intents wired for "Sync Now"; BGTaskScheduler hooks. *(observed fact)* |
| iOS BGTaskScheduler pool | `ios/Runner/Info.plist` `BGTaskSchedulerPermittedIdentifiers` | inbound | **101 identifiers** registered: `scheduled_sync_set`, `scheduled_sync_0..scheduled_sync_100`, plus `dev.flutter.background.refresh` and `be.tramckrijte.workmanagerExample.iOSBackgroundAppRefresh`. **Strong inference: per-repo container index 0..100 maps directly into a BGTaskScheduler ID slot, capping the number of containers at ~101 on iOS.** *(strong inference / portability hazard — Info.plist lines 5-108)* |
| iOS deep link | `ios/Runner/Info.plist` `CFBundleURLSchemes` = `gitsync` | inbound | Custom URL scheme handles OAuth callback (mirrors Android's `flutter_web_auth_2.CallbackActivity`). *(observed fact)* |
| iOS shortcut / Siri automation | `SyncNowIntent.swift` (625B), README "From an iOS shortcut or automation" | inbound | App Intent surfaced to Shortcuts.app. *(observed fact)* |
| iOS Document Browser / file sharing | `Info.plist` `LSSupportsOpeningDocumentsInPlace`, `UIFileSharingEnabled`, `UISupportsDocumentBrowser` all `true`; `NSPhotoLibraryUsageDescription` present | inbound | Repository directories are user-selected via document picker (forked `ios_document_picker`). *(observed fact)* |

### Network surfaces (outbound)

- **Git protocol** to user-configured remotes via libgit2 (`rust/src/api/git_manager.rs`). HTTPS Basic, SSH (key + optional encrypted-key passphrase), or OAuth-mediated HTTPS. *(observed fact — `pubspec.yaml`, `rust/Cargo.toml`, README "Authenticate with")*.
- **Provider REST APIs** to GitHub, GitLab, Gitea, Codeberg via `lib/api/manager/auth/*` (issues, PRs, comments, reactions, workflow runs, releases, tags). Uses `http` 1.6.0. *(observed fact)*.
- **AI provider HTTPS / SSE** via `lib/api/ai_stream_client.dart` (12KB) and `ai_chat_service.dart` (23KB). Provider keys stored in secure storage. *(observed fact)*.
- **OAuth callback** `gitsync://auth` redirected to via `oauth2_client` 4.2.5; both Android (`flutter_web_auth_2.CallbackActivity`) and iOS (URL scheme) handle the redirect. *(observed fact)*.

### Persistent artifacts

- **Repository working trees** at user-selected on-device paths (one per container). Listed in storage under `repoman_repoNames` and indexed via `repoman_repoIndex`. *(observed fact — `lib/api/manager/repo_manager.dart`, `lib/global.dart`)*.
- **`.git/` directories** owned by libgit2; standard packfiles, refs, config, info/exclude.
- **App settings** in SharedPreferences (Dart-side `Storage` base class, `lib/api/manager/storage.dart`).
- **OAuth tokens, SSH keys, AI provider keys** in `flutter_secure_storage` 10.0.0 (Keychain on iOS, EncryptedSharedPreferences on Android). *(observed fact — `pubspec.yaml`)*.
- **Logs** via `mixin_logger` and the local `Logger` class (`lib/api/logger.dart`).
- **Encrypted backup archives** via `archive` 4.0.9 + `cryptography` 2.9.0 + `encrypt` 5.0.1 (export/import via `enter_backup_restore_password.dart`). *(strong inference — dependency presence + dialog name)*.
- **Home-widget SharedPreferences keys** (e.g. `forceSyncWidget_status`).
- **`mmap2`-backed memory-mapped files** somewhere in the read-path — likely diff/large-file viewing. *(open question — exact use site needs `mmap2`-call site grep, deferred to defect-scan-mechanical)*.

## Runtime Lifecycle

### Cold start

1. Platform shell launches Flutter engine.
   - **Android:** `MainActivity` (singleTop, theme=`@style/LaunchTheme`, configChanges absorbed at activity level for orientation/keyboard/locale).
   - **iOS:** `AppDelegate.swift` boots the Flutter engine; `Info.plist` `UILaunchStoryboardName=LaunchScreen` shows splash.
2. `lib/main.dart:main()` runs.
   - Initializes Riverpod root; binds platform-channel handlers for `flutter_background_service`, `flutter_local_notifications`, `home_widget`, `workmanager`, `quick_actions`, `flutter_web_auth_2`. *(strong inference — file is 290KB and `lib/global.dart` requires `gitSyncService` initialised before any UI; full enumeration deferred to contracts phase via carry_forward)*.
   - Reads `lib/global.dart:demo` const (currently `false`; comment "TODO: Must be false for release"). *(observed fact)*.
   - Initialises l10n delegate (`AppLocalizations`) and stores it into `lib/global.dart:t`. *(strong inference — `lib/global.dart` line 22)*.
   - Spawns the background service via `GitsyncService.initialise(onServiceStart, callbackDispatcher)` (`lib/gitsync_service.dart:153-167`); on Android `autoStart=true, isForegroundMode=false`; on iOS the same `onServiceStart` is registered for both `onForeground` and `onBackground`.
3. UI shell renders. If `repoManager` reports first-run state (`repoman_onboardingStep == 0`), routes into `onboarding_setup.dart`. *(strong inference — `repo_manager.dart:setOnboardingStep`)*.

### Warm sync (the central loop)

The function `lib/gitsync_service.dart:_sync(repoIndex, forced, syncMessage, retryCount)` is the heart of the runtime. It is invoked by every sync trigger and runs roughly:

1. Increment `_syncGeneration` (used to detect stale finalisers).
2. Set `isSyncing=true`, push `'syncing'` state to the home widget (`HomeWidget.saveWidgetData` + `HomeWidget.updateWidget`).
3. Reload settings for `repoIndex` (`SettingsManager().reinit(repoIndex)`).
4. Verify there is a configured remote (`GitManager.listRemotes`) and credentials (HTTPS or SSH per provider).
5. Bail out if there's an in-progress merge conflict (`GitManager.getConflicting`).
6. If `optimisedSyncExperimental` flag is on, call `GitManager.getRecommendedAction(priority: 3)` to short-circuit when neither side has changes (action `-1`).
7. **Pull phase** — `GitManager.backgroundDownloadChanges` returning a tri-state `(true | false | null)` for (changes-pulled / no-pull-needed / failure).
8. Re-check conflicts (pull may have produced conflicts → bail to merge-conflict UI).
9. **Push phase** (gated by `optimisedSyncFlag` and recommendedAction in [2,3]) — `GitManager.backgroundUploadChanges` with the same tri-state.
10. Refresh `getRecentCommits()` for the UI cache.
11. On exception, route to `handleIfNetworkError(...)` which schedules a retry via `connectivity_plus` and falls back to a logged exception otherwise.
12. **Finally** (always): set `isSyncing=false`, finalize widget state via `_finishWidget(terminal)`. **iOS finalises inline with a 2-second `await Future.delayed`** because the widget-callback isolate tears down when its callback returns; **Android uses a `Timer`** because its background isolate persists. *(observed fact — `lib/gitsync_service.dart:113-138, 232-246`)*. **This is a portability hazard** — any port must distinguish callback-isolate lifetimes per platform.
13. If a sync was requested mid-flight (`isScheduled`), recurse via `debouncedSync(repoIndex)`.

### Sync triggers (sources that call into `_sync` / `debouncedSync`)

| Trigger | Owner | Path |
|---|---|---|
| App-open / app-close detected | Android `GitSyncAccessibilityService` | accessibility event → `lib/gitsync_service.dart:accessibilityEvent` → per-repo `debouncedSync(index)` if `setman_syncOnAppOpened` / `setman_syncOnAppClosed` is set |
| Scheduled sync (Android) | `workmanager` | callback dispatcher (registered in `main.dart`) → `debouncedSync` |
| Scheduled sync (iOS) | `BGTaskScheduler` (one of `scheduled_sync_0..100`) | App Intents in `BackgroundIntent.swift` → Flutter callback → `debouncedSync` *(strong inference, exact channel deferred to protocols phase)* |
| Quick Settings tile | `GitTileSyncService` / `GitTileManualSyncService` | broadcast → `INTENT_SYNC` → flutter_background_service → `gitsync_service` event constants `TILE_SYNC` / `INTENT_SYNC` / `FORCE_SYNC` / `MANUAL_SYNC` |
| Home widget tap | Android `ForceSyncWidget` / `ManualSyncWidget` receivers, iOS `ForceSyncWidget` (kind matches `_widgetIOSName`) | home_widget background callback → `_sync` (iOS inline, Android Timer) |
| `gitsync://` deep link (OAuth callback) | `flutter_web_auth_2.CallbackActivity` (Android), URL scheme (iOS) | not a sync trigger — completes OAuth, then user-initiated UI sync |
| iOS shortcut | `SyncNowIntent.swift` | App Intent → Flutter → `_sync` |
| Custom intent (Tasker etc.) | `<action android:name="INTENT_SYNC">` on `GitSyncService` | external broadcast → service → flutter background service event |
| User tap "Sync Now" in UI | `manual_sync.dart` and ad-hoc page buttons | direct `gitSyncService.debouncedSync(...)` |
| Merge complete | UI completes manual conflict resolution | `gitSyncService.merge(repoIndex, msg, conflictingPaths)` then re-syncs |

### Shutdown

No explicit application-level shutdown logic surfaced in this phase. Background services are torn down by the OS lifecycle (Android task removal, iOS background-task expiration). Foreground sync sets `isSyncing=false` and `_syncGeneration` so any pending widget revert observes "stale" and no-ops. *(strong inference)*.

## Concurrency Model

### Rust core

- `tokio` 1 with **only `rt` and `time` features** in `rust/Cargo.toml`. This means a single-threaded current-thread runtime is the default; no multi-threaded scheduler. The crate exposes git operations to FRB which marshals them via FRB's worker-pool model. *(observed fact — `rust/Cargo.toml` line 13)*.
- libgit2/git2 calls are blocking by nature; long operations (clone, fetch, push) are dispatched to FRB's background-isolate-equivalent (an FFI worker thread), not the Dart UI isolate. *(strong inference — flutter_rust_bridge default behavior; deferred to protocols phase for verification)*.

### Dart side

- Flutter UI runs on the main Dart isolate.
- `flutter_background_service` spawns a separate background isolate where `gitsync_service.dart:_sync` runs (per its README + the `Workmanager().initialize(callbackDispatcher, ...)` call). *(observed fact — `lib/gitsync_service.dart:155-167`)*.
- `home_widget` background callbacks run in **yet another isolate** that tears down per callback on iOS but persists on Android. *(observed fact — `_finishWidget` Platform-branched code in lines 113-138)*.
- AI streaming uses Dart `async`/`await` over an `http.Client` SSE stream; lives on whichever isolate started it (UI or service). *(strong inference — `ai_stream_client.dart` size and name)*.

### Synchronisation primitives

- `_syncGeneration` (`int`) + `isSyncing` (`bool`) + `isScheduled` (`bool`) form an ad-hoc reentrancy gate. **Reads/writes happen across isolates via `flutter_background_service` event-passing**, not shared memory — so the gate works only because each isolate owns its own copy and coordinates by event. *(strong inference — `lib/gitsync_service.dart:111, 142-159, 247-259`)*.
- Per-repo debouncing via `debounce(key, ms, fn)` (utility in `lib/api/helper.dart`). *(observed fact — `gitsync_service.dart:174`)*.
- Riverpod's reactive graph for UI state. Single-threaded by virtue of Dart isolate model. *(strong inference)*.
- **mmap2-flutter** (forked) is presumably used for memory-mapped reads of large files (diff viewer or repo files). Concurrency hazards around mmap span (concurrent writes from libgit2 vs. mmap-backed reads from UI) are **open question — needs runtime test or call-site review, deferred to protocols/defect-scan-semantic**.

### Backpressure / rate limiting

- Sync debounce (500ms; `gitsync_service.dart:174`).
- Network retry via `handleIfNetworkError` (lib/api/helper.dart) using `connectivity_plus` to wait for connectivity. *(observed fact — README "Retry automatically when the network returns")*.
- No explicit rate-limit on provider-API clients surfaced — likely lean on stock `http` + provider-side rate limit headers. *(open question — `arch-OQ1`)*.

### Portability hazards (concurrency)

- *(portability hazard)* iOS-vs-Android divergence in **callback-isolate lifetime** (`_finishWidget`). Hard-coded into the sync widget revert logic; a port must replicate this branching.
- *(portability hazard)* Per-repo BGTaskScheduler ID slots (`scheduled_sync_0..100`) form a **101-cap on iOS containers**. Android's `workmanager` has no equivalent fixed pool; a port must reconcile.
- *(portability hazard)* `tokio` rt+time only — switching to a different runtime model (or to Kotlin/Swift native) changes how libgit2 callback threads are pinned. The git2-rs callbacks (push/fetch progress, credentials) **must be invoked from an FFI-safe thread**.
- *(portability hazard)* `flutter_background_service` is forked at ViscousPot/flutter_background_service_with_intents; the upstream package's intent semantics differ.

## Build and Packaging

- **Flutter SDK** pinned to **3.35.2** (`.fvmrc`); Dart SDK **3.9.0** (`pubspec.yaml`); FVM-managed.
- **Rust** stable (CI uses `1.88.0`); workspace at `rust/`. `Cargo.toml` declares `crate-type = ["cdylib", "staticlib"]`. Default features: `vendored` → `git2/vendored-libgit2`, `git2/vendored-openssl`. Optional `zlib-ng-compat` for `libssh2-sys`.
- **`flutter_rust_bridge_codegen` 2.12.0** generates the FFI layer (`lib/src/rust/`, `rust/src/frb_generated.rs`). The generator is invoked via `cargo install flutter_rust_bridge_codegen` and then `flutter_rust_bridge_codegen generate`.
- **`cargokit`** (`rust/cargokit.yaml`) hooks the Cargo build into the Flutter Gradle/Xcode pipelines so `flutter build` cross-compiles the Rust crate per ABI automatically.
- **Android targets**: aarch64, armv7, x86_64, i686 (per README). minSdk 21. CI builds APKs split-per-ABI for arm, arm64, x64 in three separate `flutter build apk --release` invocations to mirror F-Droid's per-ABI metadata jobs.
- **iOS targets**: aarch64-apple-ios, aarch64-apple-ios-sim, x86_64-apple-ios. Xcode 15+ + CocoaPods.
- **Build secrets**: `lib/constant/secrets.dart` is git-ignored and templated. The CI workflow `generate-apk-release.yml` injects it from `${{ secrets.SECRETS }}` via heredoc cat. **Hazard**: secrets land at `lib/constant/secrets.dart` — anything that reads `secrets.dart` is a candidate for the security pass. *(observed fact — `.github/workflows/generate-apk-release.yml:35-39`)*.
- **CI**: single workflow `Release OSS (APK)` triggered by tags `v*` or manual dispatch. Builds, signs (using `RELEASE_KEYSTORE_BASE64` / `RELEASE_SIGNING_ALIAS` / `RELEASE_SIGNING_PASSWORD`), uploads to GitHub Releases as draft. **Note: APKs include reproducible-build tweaks** (`--remap-path-prefix=...`, `SOURCE_DATE_EPOCH` from `git log -1 --pretty=%ct`, `-Ccodegen-units=1`). **iOS releases are not in CI** — built locally / via fastlane. *(observed fact — `.github/workflows/generate-apk-release.yml:67-86`; `fastlane/metadata/android/` exists but no fastlane workflow file)*.
- **Distribution**: Play Store (`com.viscouspot.gitsync`), App Store (`id6744980427`), Izzy On Droid, F-Droid, Obtainium. Per-store metadata under `fastlane/metadata/android/`.
- **Linting**: `flutter_lints` 6 + custom `analysis_options.yaml` with `implicit-casts: false`, `implicit-dynamic: false` (strict-style Dart). Page width 150. *(observed fact)*.

A more detailed build/CI breakdown is appended to `findings/build-and-deploy/build-and-deploy.md` (secondary output, append mode).

## Porting Priorities

| Component | Priority | Rationale |
|---|---|---|
| Rust git core (`rust/src/api/git_manager.rs` + `git2`) | core | Without git operations there is no app. Single 165KB file; biggest porting unit by far. |
| `lib/gitsync_service.dart` orchestrator | core | The sync state machine. Every trigger funnels here. *(observed fact)* |
| `SettingsManager` / `RepoManager` / `Storage` (`lib/api/manager/{settings,repo}_manager.dart`, `storage.dart`) | core | Owns the per-repo container model and SharedPreferences keys (which are part of the contract — settings keys appear in widget metadata, intent extras, etc.). |
| Auth: HTTPS + SSH paths (Rust-side `git_manager.rs` + `lib/ui/component/{https,ssh}_auth_form.dart`) | core | Without auth the user cannot connect to a remote. OAuth is *important*, not core (HTTPS basic still works). |
| Trigger surfaces: accessibility-driven sync, scheduled sync, quick tile, home widget, custom intent | important | Major value of the app over a stock git client. iOS-only target may collapse some (no quick tile or accessibility), but home widget + scheduled sync remain. |
| Provider clients (GitHub/GitLab/Gitea/Codeberg) | important | Issues/PRs/Actions/releases are differentiating features. Three large managers (170KB total) — likely the second-biggest porting unit. |
| AI features (chat, autocomplete, agent, tools) | important | Marketed feature; gated by global toggle. Could be deferred for a v1 port. |
| Code editor / file explorer / diff view | important | In-app editing is a marketed differentiator but a port can reuse a library editor. |
| Onboarding wizard | optional | Single-file page — recreatable. |
| Backup/restore (encrypted archive) | optional | Convenience feature. |
| Showcase/feature-tutorial overlays | incidental | Polish. |
| Maestro UI test harness (`.maestro/`) | incidental | Tooling, not product. Can be reimplemented in any UI test framework. |
| Forked package patches (flutter_background_service intent extensions, mmap2 fork, ios_document_picker fork) | important *(portability hazard)* | These forks exist because upstream behavior was insufficient. A port must replicate the diffs OR pick a stack where the underlying capability exists natively. |
| Provider OAuth secrets ingestion (CI heredoc into `lib/constant/secrets.dart`) | important | The build process is part of the contract — any port must continue to inject OAuth client IDs at build time. |

## Durable State

| Kind | Storage | Notes |
|---|---|---|
| Per-app settings | SharedPreferences via `lib/api/manager/storage.dart`/`settings_manager.dart` | Prefix `setman_*` (e.g. `setman_syncOnAppClosed`, `setman_optimisedSyncExperimental`, `setman_syncMessageEnabled`). *(observed fact)* |
| Per-repo container index | SharedPreferences key `repoman_repoIndex`; names list `repoman_repoNames`; onboarding step `repoman_onboardingStep`; namespace `git_sync_repos` | *(observed fact — `lib/api/manager/repo_manager.dart`)* |
| OAuth tokens (GitHub / GitLab / Gitea / Codeberg) | `flutter_secure_storage` 10.0.0 | Keychain (iOS) / EncryptedSharedPreferences (Android). |
| HTTPS basic credentials | secure storage (per-repo) | `getGitHttpAuthCredentials()` returns `(username, password)`; bail-out if password empty. |
| SSH private keys + optional passphrases | secure storage (per-repo) | `getGitSshAuthCredentials()`; supports encrypted keys via `osshkeys`. |
| AI provider API keys | secure storage | `aiKeyConfigured` ValueNotifier reflects presence. |
| Application packages list (for accessibility-triggered sync) | per-repo settings | `settingsManager.getApplicationPackages()` |
| Home-widget status | SharedPreferences key `forceSyncWidget_status` | Values `'syncing'` / `'idle'` / `'success'` / `'error'`. |
| Repository working trees | filesystem at user-selected path | Owned by libgit2; `.git/` packfiles under it. |
| Logs | local files via `Logger` (`lib/api/logger.dart`) + `mixin_logger` | |
| Encrypted backup archives | filesystem (user-chosen path) | Built with `archive` + `cryptography` + `encrypt`. |
| iOS BGTaskScheduler-registered identifiers | `Info.plist` static (101 IDs) | Hard-coded; not generated. |
| `secrets.dart` (OAuth client IDs/secrets) | source-tree, git-ignored, baked at build time | CI injects from `secrets.SECRETS`. Stored in compiled binary. |

## Open Questions

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| `arch-OQ1` | needs-runtime-test | Is there an explicit rate-limit / backoff on provider-API clients (GitHub/GitLab/Gitea), or do they just rely on `http` + provider-side 429 responses? | The auth/manager files are 47-70KB each; a focused contract/protocol pass will surface this without a runtime test if the source is grep-able for retry patterns. Routed below as `arch-CF1` to contracts. |
| `arch-OQ2` | needs-runtime-test | Are AI provider responses ever cached, or always live SSE? | `ai_stream_client.dart` is 12KB; the contracts phase will surface caching by reading the file. Routed as `arch-CF2`. |
| `arch-OQ3` | needs-maintainer-decision | Why is `lib/main.dart` 290KB single-file? Is this an intentional bundling for tree-shaking, an artifact of FRB callback registration needing one site, or just historical? | A maintainer may want to refactor; either way the contracts phase needs to walk it page-by-page. |

## Carry-Forward

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| `arch-CF1` | contracts | Catalog of provider-API endpoints invoked by `github_manager.dart` (70KB), `gitlab_manager.dart` (47KB), `gitea_manager.dart` (49KB), `codeberg_manager.dart` (small wrapper); pin retry/rate-limit behavior. | The behavioral-contracts phase walks user-visible features feature-by-feature and will naturally enumerate REST endpoints + retry semantics. |
| `arch-CF2` | contracts | AI feature contract — chat/wand/agent flow trigger -> side effect -> persisted state -> error behavior. Uses `ai_stream_client.dart` and 11 files in `lib/api/ai_*.dart`. | Contracts phase walks user-visible features; AI tab is a major feature surface. |
| `arch-CF3` | protocols | flutter_rust_bridge wire surface: enumerate every Rust-side function exposed via FRB and pair with its Dart caller. `rust/src/api/git_manager.rs` is 165KB; `lib/api/manager/git_manager.dart` is 53KB. | Protocols phase rubric is exactly this kind of wire-format extraction. |
| `arch-CF4` | protocols | `flutter_background_service_with_intents` (forked) intent broadcast protocol — what intents the service accepts, what data they carry, what it broadcasts back. Constants visible in `lib/gitsync_service.dart`. | Protocols phase. |
| `arch-CF5` | protocols | `home_widget` IPC protocol — keys shared via SharedPreferences (`forceSyncWidget_status`), `qualifiedAndroidName` resolution to a Receiver, iOS `WidgetCenter.shared.reloadTimelines(ofKind:)` matching `ForceSyncWidget`. | Protocols phase. |
| `arch-CF6` | protocols | iOS BGTaskScheduler 101-ID pool semantics — how slots are assigned to repos, what happens past the 100th container, register/unregister lifecycle. | Protocols phase. |
| `arch-CF7` | defect-scan-mechanical | `lib/global.dart:demo` const flag (`TODO: Must be false for release`) — release-state hazard if it ever flips inadvertently. | Mechanical defect rubric covers config/environment. |
| `arch-CF8` | defect-scan-mechanical | `mmap2_flutter` (forked) usage call-sites — concurrency hazards if libgit2 writes overlap with UI mmap reads. | Mechanical pass enumerates dangerous patterns. |
| `arch-CF9` | defect-scan-semantic | iOS-vs-Android `_finishWidget` Timer-vs-inline split — verify gen-counter race correctness across both isolate-lifetime models. | Concurrency pass. |
| `arch-CF10` | defect-scan-semantic | `secrets.dart` runtime use — verify no path leaks the OAuth client secret into logs, errors, or telemetry. | Security pass. |

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | The system intent is documented. | PASS | §System Intent. Single paragraph naming target users, problem solved, and concrete app shape. |
| 2 | The layer map and dependency direction are documented. | PASS | §Layer Map -> Package Inventory (17 entries) + Dependency Direction (ASCII graph + 5 directional observations). No cycles found at package level; `lib/main.dart` flagged as a structural cycle risk. |
| 3 | Public surfaces are identified. | PASS | §Public Surfaces split into Mobile UI (12 entries), OS-integration (12 entries with intent filters and BGTaskScheduler IDs), Network (4 outbound surfaces), and Persistent artifacts (12 kinds). |
| 4 | Runtime lifecycle, concurrency model, and porting priorities are summarized. | PASS | §Runtime Lifecycle (cold start + warm-sync state machine + 9-row trigger table + shutdown). §Concurrency Model (Rust tokio rt+time; Dart isolates; sync primitives; portability hazards). §Porting Priorities (13-row table with priority + rationale). |
| 5 | Findings are marked with evidence levels. | PASS | Every load-bearing claim is tagged `observed fact`, `strong inference`, `portability hazard`, or `open question` inline. Open questions pinned to §Open Questions (3 entries) and routed via §Carry-Forward (10 entries) into specific later phases. |

**Validated by:** 2026-05-06 (architecture phase, implementing session)
**Overall:** PASS
