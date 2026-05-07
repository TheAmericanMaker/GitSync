# GitSync Source File Inventory

Generated 2026-05-06 for the architecture phase. Walks lib/, rust/src/, manifests, platform shells, tests, and assets. Auto-generated artifacts (lib/src/rust/, rust/src/frb_generated.rs ~209KB) are excluded.

## Top-level manifests and config

| Path | Size | Purpose |
| --- | --- | --- |
| pubspec.yaml | 3.4KB | Flutter app manifest (deps, fonts, assets, plugins). |
| pubspec.lock | 38.9KB | Resolved Dart dependency lockfile. |
| analysis_options.yaml | 1.5KB | Dart analyzer / lint configuration. |
| flutter_rust_bridge.yaml | 65B | flutter_rust_bridge codegen config (Dart <-> Rust FFI). |
| flutter_launcher_icons.yaml | 277B | flutter_launcher_icons plugin config. |
| l10n.yaml | 165B | Flutter intl/l10n generation config. |
| .fvmrc | 25B | FVM (Flutter Version Manager) pinned SDK version. |
| fvm_config.json | 36B | Legacy FVM config. |
| .metadata | 1.7KB | Flutter project tracking metadata. |
| README.md | 8.8KB | Project README. |
| LICENSE.md | 34.1KB | License file (GPL-style, large). |
| untranslated.txt | 88.9KB | List of untranslated l10n strings. |
| flutter_01.png | 0B | Empty placeholder PNG (zero bytes). |

## Dart sources (lib/)

Note: `lib/src/rust/` (flutter_rust_bridge auto-generated bindings) excluded per scope.

### lib/ (root)
| Path | Size | Purpose |
| --- | --- | --- |
| lib/main.dart | 289.8KB | App entry point, UI shell, navigation, top-level widget tree (monolith). |
| lib/gitsync_service.dart | 17.5KB | Core GitSync service orchestration (sync engine wrapper). |
| lib/global.dart | 854B | Global singletons / app-wide state holder. |

### lib/api/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/api/accessibility_service_helper.dart | 3.2KB | Bridge to Android accessibility service for auto-sync triggers. |
| lib/api/ai_chat_service.dart | 23.0KB | AI chat service (LLM conversation handler). |
| lib/api/ai_completion_service.dart | 4.4KB | AI completion / commit message generation. |
| lib/api/ai_provider_validator.dart | 4.9KB | Validates AI provider credentials/config. |
| lib/api/ai_stream_client.dart | 11.8KB | Streaming HTTP client for LLM responses (SSE). |
| lib/api/ai_system_prompt.dart | 10.3KB | AI system prompt templates. |
| lib/api/ai_tool_executor.dart | 1.8KB | Dispatches AI tool calls to executors. |
| lib/api/ai_tools.dart | 3.2KB | AI tool definitions registry. |
| lib/api/ai_tools_file.dart | 11.2KB | AI tools for file operations (read/write/list). |
| lib/api/ai_tools_git.dart | 44.7KB | AI tools for git operations (commit, branch, diff, etc). |
| lib/api/ai_tools_provider.dart | 33.8KB | AI tools for git-provider operations (issues, PRs, releases). |
| lib/api/colour_provider.dart | 3.1KB | Theme/color state provider. |
| lib/api/helper.dart | 28.7KB | Cross-cutting helpers / utility grab-bag. |
| lib/api/issue_template_parser.dart | 5.6KB | Parses GitHub issue template YAML/markdown. |
| lib/api/logger.dart | 11.8KB | Logging facility. |

### lib/api/manager/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/api/manager/git_manager.dart | 53.2KB | High-level Dart git manager (calls Rust core via FFI). |
| lib/api/manager/premium_manager.dart | 3.0KB | Premium/IAP entitlement manager. |
| lib/api/manager/repo_manager.dart | 983B | Repository registry / persistence. |
| lib/api/manager/settings_manager.dart | 7.9KB | App settings persistence. |
| lib/api/manager/storage.dart | 10.5KB | Storage abstraction (SharedPreferences/secure storage). |

### lib/api/manager/auth/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/api/manager/auth/codeberg_manager.dart | 237B | Codeberg auth/API thin wrapper. |
| lib/api/manager/auth/git_provider_manager.dart | 8.8KB | Generic git-provider abstraction (multiplexer). |
| lib/api/manager/auth/gitea_manager.dart | 49.0KB | Gitea API client (auth, issues, PRs, repos). |
| lib/api/manager/auth/github_app_manager.dart | 1.1KB | GitHub App auth specifics. |
| lib/api/manager/auth/github_manager.dart | 70.1KB | GitHub API client (auth, issues, PRs, repos). |
| lib/api/manager/auth/gitlab_manager.dart | 47.1KB | GitLab API client (auth, issues, MRs, repos). |

### lib/constant/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/constant/dimens.dart | 1.2KB | Layout dimension constants. |
| lib/constant/icons.dart | 426B | Icon constants. |
| lib/constant/langDiff.dart | 931B | Language-specific diff config. |
| lib/constant/langLog.dart | 1.1KB | Language-specific log strings. |
| lib/constant/reactions.dart | 1.2KB | GitHub reaction emoji constants. |
| lib/constant/secrets.dart.template | 233B | Template for compile-time secrets (API keys). |
| lib/constant/strings.dart | 10.9KB | Static string constants. |
| lib/constant/values.dart | 3.9KB | Misc. constant values. |

### lib/l10n/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/l10n/app.de.arb | 19.4KB | German localization (legacy filename). |
| lib/l10n/app_ar.arb | 48.9KB | Arabic localization. |
| lib/l10n/app_en.arb | 42.5KB | English localization (source-of-truth). |
| lib/l10n/app_es.arb | 14.0KB | Spanish localization. |
| lib/l10n/app_fr.arb | 18.5KB | French localization. |
| lib/l10n/app_ja.arb | 33.3KB | Japanese localization. |
| lib/l10n/app_ru.arb | 20.0KB | Russian localization. |
| lib/l10n/app_zh.arb | 27.3KB | Chinese (Simplified) localization. |
| lib/l10n/app_zh_Hant.arb | 151B | Chinese (Traditional) stub. |
| lib/l10n/app_localizations.dart | 134.9KB | Generated l10n delegate (root). |
| lib/l10n/app_localizations_ar.dart | 65.0KB | Generated l10n (ar). |
| lib/l10n/app_localizations_de.dart | 60.2KB | Generated l10n (de). |
| lib/l10n/app_localizations_en.dart | 58.5KB | Generated l10n (en). |
| lib/l10n/app_localizations_es.dart | 59.6KB | Generated l10n (es). |
| lib/l10n/app_localizations_fr.dart | 61.0KB | Generated l10n (fr). |
| lib/l10n/app_localizations_ja.dart | 63.4KB | Generated l10n (ja). |
| lib/l10n/app_localizations_ru.dart | 65.7KB | Generated l10n (ru). |
| lib/l10n/app_localizations_zh.dart | 57.6KB | Generated l10n (zh). |
| lib/l10n/fix_order.sh | 2.9KB | Helper bash script to reorder ARB keys. |

### lib/providers/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/providers/riverpod_providers.dart | 21.0KB | Centralized Riverpod providers (state mgmt). |

### lib/type/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/type/action_run.dart | 579B | Type: GitHub Actions workflow run. |
| lib/type/ai_chat.dart | 3.1KB | Type: AI chat message / conversation. |
| lib/type/git_provider.dart | 387B | Type: enum/struct for git provider kind. |
| lib/type/issue.dart | 948B | Type: issue list entry. |
| lib/type/issue_detail.dart | 2.4KB | Type: detailed issue model. |
| lib/type/issue_template.dart | 1.5KB | Type: issue template schema. |
| lib/type/pr_detail.dart | 5.5KB | Type: detailed PR model. |
| lib/type/pull_request.dart | 757B | Type: PR list entry. |
| lib/type/release.dart | 757B | Type: release entry. |
| lib/type/showcase_feature.dart | 2.0KB | Type: feature-showcase descriptor. |
| lib/type/tag.dart | 251B | Type: git tag. |

### lib/ui/component/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/ui/component/ai_wand_field.dart | 4.3KB | "AI wand" text field with quick-fill. |
| lib/ui/component/auto_sync_settings.dart | 18.7KB | Auto-sync settings UI block. |
| lib/ui/component/branch_selector.dart | 12.3KB | Branch picker widget. |
| lib/ui/component/button_setting.dart | 5.6KB | Button-style settings row. |
| lib/ui/component/code_line_number_render_object.dart | 10.4KB | Custom RenderObject for code line numbers. |
| lib/ui/component/commit_select_action_bar.dart | 10.9KB | Action bar for multi-select on commits list. |
| lib/ui/component/custom_showcase.dart | 6.6KB | Custom feature-showcase overlay widget. |
| lib/ui/component/diff_file.dart | 20.2KB | File diff viewer widget. |
| lib/ui/component/group_sync_settings.dart | 2.0KB | Grouped sync settings container. |
| lib/ui/component/html_markdown.dart | 6.2KB | HTML/markdown renderer widget. |
| lib/ui/component/https_auth_form.dart | 6.4KB | HTTPS auth form (username/password/token). |
| lib/ui/component/item_commit.dart | 21.6KB | Commit list item widget. |
| lib/ui/component/item_merge_conflict.dart | 4.1KB | Merge conflict list item. |
| lib/ui/component/item_setting.dart | 5.0KB | Generic settings list item. |
| lib/ui/component/markdown_config.dart | 4.0KB | Shared markdown styling config. |
| lib/ui/component/post_footer_indicator.dart | 1.3KB | Post-footer loading indicator. |
| lib/ui/component/provider_builder.dart | 484B | Riverpod provider Consumer helper. |
| lib/ui/component/quick_sync_settings.dart | 14.6KB | Quick-sync settings block. |
| lib/ui/component/scheduled_sync_settings.dart | 20.5KB | Scheduled-sync settings block. |
| lib/ui/component/showcase_feature_button.dart | 8.4KB | Feature-showcase button. |
| lib/ui/component/ssh_auth_form.dart | 11.3KB | SSH auth form. |
| lib/ui/component/sync_client_mode_toggle.dart | 11.0KB | Toggle for sync client mode. |
| lib/ui/component/sync_loader.dart | 10.8KB | Sync progress loader animation. |

### lib/ui/dialog/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/ui/dialog/add_container.dart | 2.8KB | Dialog: add repo container. |
| lib/ui/dialog/add_remote.dart | 6.4KB | Dialog: add git remote. |
| lib/ui/dialog/amend_commit.dart | 5.0KB | Dialog: amend commit. |
| lib/ui/dialog/auth.dart | 22.3KB | Dialog: authentication flow root. |
| lib/ui/dialog/author_details_prompt.dart | 1.7KB | Dialog: prompt for author name/email. |
| lib/ui/dialog/base_alert_dialog.dart | 16.8KB | Base alert dialog scaffold. |
| lib/ui/dialog/change_language.dart | 3.0KB | Dialog: change app language. |
| lib/ui/dialog/cloning_repository.dart | 3.1KB | Dialog: clone-in-progress. |
| lib/ui/dialog/confirm_branch_checkout.dart | 2.9KB | Confirm: branch checkout. |
| lib/ui/dialog/confirm_checkout_commit.dart | 3.4KB | Confirm: checkout specific commit. |
| lib/ui/dialog/confirm_cherry_pick.dart | 6.3KB | Confirm: cherry-pick. |
| lib/ui/dialog/confirm_clear_data.dart | 1.9KB | Confirm: clear data. |
| lib/ui/dialog/confirm_clone_overwrite.dart | 2.8KB | Confirm: overwrite on clone. |
| lib/ui/dialog/confirm_delete_branch.dart | 1.9KB | Confirm: delete branch. |
| lib/ui/dialog/confirm_delete_file_folder.dart | 2.4KB | Confirm: delete file/folder. |
| lib/ui/dialog/confirm_delete_remote.dart | 1.9KB | Confirm: delete remote. |
| lib/ui/dialog/confirm_discard_changes.dart | 2.1KB | Confirm: discard changes. |
| lib/ui/dialog/confirm_force_push_pull.dart | 1.9KB | Confirm: force push/pull. |
| lib/ui/dialog/confirm_multi_cherry_pick.dart | 7.2KB | Confirm: multi-cherry-pick. |
| lib/ui/dialog/confirm_priv_key_copy.dart | 1.6KB | Confirm: copy SSH private key. |
| lib/ui/dialog/confirm_reinstall_clear_data.dart | 2.8KB | Confirm: reinstall + clear. |
| lib/ui/dialog/confirm_remove_container.dart | 1.9KB | Confirm: remove repo container. |
| lib/ui/dialog/confirm_reset_to_commit.dart | 3.1KB | Confirm: reset to commit. |
| lib/ui/dialog/confirm_revert_commit.dart | 3.1KB | Confirm: revert commit. |
| lib/ui/dialog/confirm_squash_commits.dart | 7.2KB | Confirm: squash commits. |
| lib/ui/dialog/confirm_undo_commit.dart | 3.1KB | Confirm: undo last commit. |
| lib/ui/dialog/create_branch.dart | 6.3KB | Dialog: create branch. |
| lib/ui/dialog/create_branch_from_commit.dart | 3.4KB | Dialog: create branch from commit. |
| lib/ui/dialog/create_file.dart | 2.7KB | Dialog: create file. |
| lib/ui/dialog/create_folder.dart | 2.7KB | Dialog: create folder. |
| lib/ui/dialog/create_repository.dart | 10.4KB | Dialog: create repository. |
| lib/ui/dialog/create_tag_on_commit.dart | 3.4KB | Dialog: create tag on commit. |
| lib/ui/dialog/diff_view.dart | 23.1KB | Dialog: diff viewer. |
| lib/ui/dialog/enter_backup_restore_password.dart | 3.0KB | Dialog: backup/restore password prompt. |
| lib/ui/dialog/enter_gh_sponsor_pat.dart | 2.5KB | Dialog: enter GitHub Sponsor PAT. |
| lib/ui/dialog/error_occurred.dart | 9.3KB | Dialog: generic error display. |
| lib/ui/dialog/force_push_pull.dart | 1.6KB | Dialog: force push/pull picker. |
| lib/ui/dialog/github_issue_oauth.dart | 2.2KB | Dialog: GitHub issue OAuth flow. |
| lib/ui/dialog/github_issue_report.dart | 12.5KB | Dialog: file a GitHub issue (bug report). |
| lib/ui/dialog/github_scoped_guide.dart | 4.2KB | Dialog: GitHub scoped-PAT guide. |
| lib/ui/dialog/import_priv_key.dart | 6.5KB | Dialog: import SSH private key. |
| lib/ui/dialog/info_dialog.dart | 1.3KB | Dialog: simple info popup. |
| lib/ui/dialog/issue_reported_successfully.dart | 1.4KB | Dialog: issue submitted confirmation. |
| lib/ui/dialog/manual_sync.dart | 83.6KB | Dialog: manual sync orchestrator (large). |
| lib/ui/dialog/merge_conflict.dart | 61.2KB | Dialog: merge-conflict resolver (large). |
| lib/ui/dialog/obisidian_git_found.dart | 1.6KB | Dialog: detected obsidian-git plugin. |
| lib/ui/dialog/prominent_disclosure.dart | 1.7KB | Dialog: Play Store prominent-disclosure. |
| lib/ui/dialog/prompt_disable_ssl.dart | 1.8KB | Dialog: prompt to disable SSL verification. |
| lib/ui/dialog/remove_container.dart | 3.5KB | Dialog: remove repo container. |
| lib/ui/dialog/rename_branch.dart | 3.4KB | Dialog: rename branch. |
| lib/ui/dialog/rename_container.dart | 2.9KB | Dialog: rename repo container. |
| lib/ui/dialog/rename_file_folder.dart | 2.8KB | Dialog: rename file/folder. |
| lib/ui/dialog/rename_remote.dart | 3.4KB | Dialog: rename remote. |
| lib/ui/dialog/repo_url_invalid.dart | 1.6KB | Dialog: invalid repo URL. |
| lib/ui/dialog/select_application.dart | 10.7KB | Dialog: select Android app for auto-sync. |
| lib/ui/dialog/select_folder.dart | 7.9KB | Dialog: folder picker. |
| lib/ui/dialog/set_remote_url.dart | 2.9KB | Dialog: set remote URL. |
| lib/ui/dialog/submodules_found.dart | 1.6KB | Dialog: notice when submodules found. |

### lib/ui/page/
| Path | Size | Purpose |
| --- | --- | --- |
| lib/ui/page/actions_page.dart | 14.6KB | Page: GitHub Actions runs. |
| lib/ui/page/ai_features_page.dart | 79.8KB | Page: AI features hub (chat, completions). |
| lib/ui/page/clone_repo_main.dart | 35.7KB | Page: clone repo flow. |
| lib/ui/page/code_editor.dart | 32.4KB | Page: in-app code editor. |
| lib/ui/page/create_issue_page.dart | 30.2KB | Page: create issue. |
| lib/ui/page/create_pr_page.dart | 22.2KB | Page: create pull request. |
| lib/ui/page/expanded_commits.dart | 33.3KB | Page: expanded commits view. |
| lib/ui/page/file_explorer.dart | 48.4KB | Page: repo file explorer. |
| lib/ui/page/global_settings_main.dart | 54.0KB | Page: global app settings. |
| lib/ui/page/image_viewer.dart | 2.0KB | Page: image viewer. |
| lib/ui/page/issue_detail_page.dart | 41.3KB | Page: issue detail. |
| lib/ui/page/issues_page.dart | 35.5KB | Page: issues list. |
| lib/ui/page/onboarding_setup.dart | 158.8KB | Page: onboarding wizard (large). |
| lib/ui/page/pr_detail_page.dart | 52.0KB | Page: PR detail. |
| lib/ui/page/pull_requests_page.dart | 30.8KB | Page: PR list. |
| lib/ui/page/releases_page.dart | 18.9KB | Page: releases list. |
| lib/ui/page/settings_main.dart | 35.5KB | Page: per-repo settings. |
| lib/ui/page/sync_settings_main.dart | 2.5KB | Page: sync-settings hub. |
| lib/ui/page/tags_page.dart | 13.6KB | Page: tags list. |
| lib/ui/page/unlock_premium.dart | 30.5KB | Page: unlock premium / paywall. |

## Rust sources (rust/src/)

Note: `rust/src/frb_generated.rs` (~209KB, flutter_rust_bridge auto-generated) excluded.

### rust/src/ (root)
| Path | Size | Purpose |
| --- | --- | --- |
| rust/src/lib.rs | 32B | Crate root; declares `api` module. |

### rust/src/api/
| Path | Size | Purpose |
| --- | --- | --- |
| rust/src/api/git_manager.rs | 164.9KB | Rust core: libgit2/git2-backed git operations exposed via FFI. |
| rust/src/api/mod.rs | 35B | Module root for `api`. |

### rust/src/api/test/
| Path | Size | Purpose |
| --- | --- | --- |
| rust/src/api/test/git_manager.rs | 26.4KB | Rust unit/integration tests for git_manager. |
| rust/src/api/test/mod.rs | 20B | Module root for `api::test`. |
| rust/src/api/test/.env | 429B | Test fixture env file. |

## Tests

### test/
| Path | Size |
| --- | --- |
| test/widget_test.dart | 1.0KB |

### integration_test/
| Path | Size |
| --- | --- |
| integration_test/simple_test.dart | 526B |

### test_driver/
| Path | Size |
| --- | --- |
| test_driver/integration_test.dart | 109B |

## Platform shells

### android/app/src/main/
| Path | Size |
| --- | --- |
| android/app/src/main/AndroidManifest.xml | 7.0KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/AccessibilityServiceHelper.kt | 9.4KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/GitSyncAccessibilityService.kt | 1.4KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/GitSyncService.kt | 1.1KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/GitTileManualSyncService.kt | 1.4KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/GitTileSyncService.kt | 505B |
| android/app/src/main/kotlin/com/viscouspot/gitsync/MainActivity.kt | 1.4KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/widget/ForceSyncWidget.kt | 5.5KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/widget/ForceSyncWidgetReceiver.kt | 243B |
| android/app/src/main/kotlin/com/viscouspot/gitsync/widget/ManualSyncWidget.kt | 4.8KB |
| android/app/src/main/kotlin/com/viscouspot/gitsync/widget/ManualSyncWidgetReceiver.kt | 246B |
| android/app/src/main/res/ | (Android resources, not enumerated) |

### ios/Runner/
| Path | Size |
| --- | --- |
| ios/Runner/AppDelegate.swift | 12.9KB |
| ios/Runner/BackgroundIntent.swift | 815B |
| ios/Runner/Info.plist | 6.4KB |
| ios/Runner/Runner-Bridging-Header.h | 38B |
| ios/Runner/Runner.entitlements | 355B |
| ios/Runner/RunnerDebug.entitlements | 356B |
| ios/Runner/SyncNowIntent.swift | 625B |
| ios/Runner/Assets.xcassets/ | (asset catalog, not enumerated) |
| ios/Runner/Base.lproj/ | (storyboards, not enumerated) |

## CI / build / assets

### .github/workflows/
| File | Workflow `name:` |
| --- | --- |
| .github/workflows/generate-apk-release.yml | "Release OSS (APK)" — tags v*: build/sign/upload split-per-ABI APKs and create GitHub Release. |

### .maestro/

Maestro UI-test harness. Top-level files:

| Path | Size | Purpose |
| --- | --- | --- |
| .maestro/Cargo.toml | 163B | Cargo manifest for the maestro test runner crate. |
| .maestro/Cargo.lock | 28.8KB | Cargo lockfile. |
| .maestro/README.md | 2.4KB | Maestro test docs. |
| .maestro/config.yaml | 215B | Maestro runner config. |
| .maestro/untranslated.txt | 2B | (placeholder) |
| .maestro/build/.last_build_id | 32B | Cached build id. |

Test flow groups (each contains YAML flows + sometimes nested flow subdirs):
- `.maestro/auth/` — gitea.yaml, github.yaml, https.yaml, ssh.yaml + flows/{gitea,github,https,open_auth,ssh}/ + src/{gitea.rs, github.rs, https.rs, ssh.rs, mod.rs}
- `.maestro/auto_sync/flows/` — add_applications/, enable_accessibility_service/, prominent_disclosure_dialog/
- `.maestro/clone/flows/` — list/, local/, select_folder/, select_folder_dialog/, url/ + assert.yaml
- `.maestro/common/` — back.yaml, cleanup.yaml, clear_state.yaml, disable_all_files_access.yaml, kill.yaml, launch_notif_perm.yaml, launch_stls_no_perms.yaml
- `.maestro/generate_screenshots/` — auth.yaml, auto_sync_settings.yaml, homepage.yaml, manual_sync.yaml, merge_conflict.yaml, quick_sync_settings.yaml, scheduled_sync_settings.yaml, select_apps.yaml, settings.yaml, settings_bottom.yaml, settings_top.yaml + raw/
- `.maestro/home/flows/` — auth/, deselect_repo/, sync_messages/, sync_now/
- `.maestro/onboarding/` — negative.yaml, positive.yaml, src/{negative.rs, positive.rs, mod.rs}, flows/{all_files_dialog, all_files_permission, almost_there_dialog, author_details_prompt, legacy_user_dialog, notifications_dialog, notifications_permission, pre_auth_dialog, showcase, welcome_dialog}/
- `.maestro/settings/` — positive.yaml, flows/{author_email, author_name}/ + assert.yaml
- `.maestro/src/` — env.rs.template, generate_screenshots.rs (16.8KB), main.rs (8.6KB) — Rust runner driving Maestro flows.

Note: deeply nested per-step flow YAMLs under each flow group not individually enumerated to keep inventory compact; subtrees are predictable mirrors of the UI flow names.

### fastlane/
| Path |
| --- |
| fastlane/metadata/android/ (Play Store metadata; subtree not enumerated) |

### assets/
| Path | Size |
| --- | --- |
| assets/app_icon.png | 30.2KB |

### fonts/
| Path | Size |
| --- | --- |
| fonts/CodebergLogo.ttf | 1.9KB |
| fonts/GiteaLogo.ttf | 2.3KB |
| fonts/GitlabLogo.ttf | 1.9KB |
| fonts/AtkinsonHyperlegible/ | (font family subdir) |
| fonts/RobotoMono/ | (font family subdir) |

## Summary stats

- Total Dart source files (excluding lib/src/rust/, l10n generated, and ARB): ~127 (lib root: 3; lib/api: 14; lib/api/manager: 5; lib/api/manager/auth: 6; lib/constant: 8; lib/providers: 1; lib/type: 11; lib/ui/component: 22; lib/ui/dialog: 56; lib/ui/page: 19) — counting only hand-written Dart, not generated `app_localizations*.dart`.
- Total Rust source files (excluding frb_generated.rs): 5 (lib.rs, api/mod.rs, api/git_manager.rs, api/test/mod.rs, api/test/git_manager.rs).
- Total Dart hand-written bytes: ~2.05 MB (dominated by `main.dart` 290KB, `onboarding_setup.dart` 159KB, `manual_sync.dart` 84KB, `ai_features_page.dart` 80KB, `github_manager.dart` 70KB).
- Largest hand-written Dart files (top 5):
  1. lib/main.dart — 289.8KB
  2. lib/ui/page/onboarding_setup.dart — 158.8KB
  3. lib/ui/dialog/manual_sync.dart — 83.6KB
  4. lib/ui/page/ai_features_page.dart — 79.8KB
  5. lib/api/manager/auth/github_manager.dart — 70.1KB
- Largest Rust files (top 5):
  1. rust/src/api/git_manager.rs — 164.9KB
  2. rust/src/api/test/git_manager.rs — 26.4KB
  3. rust/src/api/mod.rs — 35B
  4. rust/src/lib.rs — 32B
  5. rust/src/api/test/mod.rs — 20B
- Test files: 3 Dart (test/widget_test.dart, integration_test/simple_test.dart, test_driver/integration_test.dart) + 2 Rust (rust/src/api/test/{mod.rs, git_manager.rs}).
- Platform shell files: 11 Android (1 manifest + 10 Kotlin) + 7 iOS (Info.plist, 4 Swift, bridging header, 2 entitlements). Resource directories not enumerated.
- CI workflows: 1 (.github/workflows/generate-apk-release.yml).
