# Closeout — 2026-05-06 — architecture

## Summary

First implementing-session pass over the deep-audit pipeline against gitsync. Produced `findings/architecture/architecture-map.md` (~40KB) anchored on a structural file inventory (`scratch/file-inventory.md`, 22KB). Identified the runtime fulcrum (`lib/gitsync_service.dart:_sync`), the lowest stable layer (`rust/src/api/git_manager.rs` over libgit2), and the load-bearing portability hazards (iOS-Android `_finishWidget` isolate-lifetime split; iOS BGTaskScheduler 101-ID pool capping containers; three forked Dart packages overriding upstream). Routed 10 carry-forward items to downstream phases by target_phase; routed one open question (`arch-OQ3` — main.dart's 290KB monolith origin) as a maintainer-level item that no pipeline phase can close. Next session enters defect-scan-mechanical with concrete targets in `next_actions`.

## Files Touched

- **Added:**
  - `.codecarto/findings/architecture/architecture-map.md`
  - `.codecarto/scratch/file-inventory.md` (subagent-produced, committed in `67fc86d9`)
  - `.codecarto/closeouts/2026-05-06-architecture.md` (this file)
- **Modified:**
  - `.codecarto/workflow/status.yaml` (architecture -> complete; current_phase advanced; carry_forward + open_questions populated)
  - `.codecarto/THREAD_LOG.md` (one new index line)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block per workflow/VALIDATE.md | PASS | All 5 architecture criteria PASS. |
| Output path matches pipeline `primary_output` | PASS | Matches `findings/architecture/architecture-map.md` per pipeline-full-with-deep-audit.yaml. |
| Evidence levels tagged on every claim | PASS | observed fact / strong inference / portability hazard / open question used throughout. |
| `dart analyze` / `cargo check` / `flutter test` | n/a | Source code is read-only in this analysis run; no code changes that would warrant invoking gitsync's gates. |

## Decisions Beyond Prompt

- **D001** | Treat `lib/src/rust/` and `rust/src/frb_generated.rs` as generated artifacts and exclude them from analysis depth (they appear only in build-and-packaging notes). | flutter_rust_bridge regenerates them on every Rust-API change; reading them in defect/contract passes wastes context for a derived view of `rust/src/api/git_manager.rs`, which is the source of truth.
- **D002** | Treat `.maestro/` (the Rust-driven Maestro UI test harness) as out-of-product tooling for purposes of architecture, contracts, and defect scans. | It is a test-runner, not shipped to users; product behavior comes from `lib/` and `rust/src/api/`. The harness is mentioned in the layer map as `tooling (excluded from product)` to keep the dependency graph honest.
- **D003** | Use the inventory subagent's table-of-files as the canonical reference for which files exist; primary phases only read individual files when they need contents. | The framework's GUIDE explicitly endorses this for codebases over ~50 files; gitsync has 127 Dart sources hand-written.

## Proposed Conventions

### C1 Inventory-driven phase scoping

**Why:** Multi-language mobile apps with monolithic entry files (gitsync's `main.dart` at 290KB) make naive deep-reads context-prohibitive. A subagent-produced file inventory (path + size + one-line purpose) lets the primary thread choose deep-read targets surgically and lets later phases re-target against the same inventory without re-walking.

**How to apply:** In phase 1, spawn one subagent whose only job is to walk the source tree and produce `scratch/file-inventory.md`. Cite its commit SHA in the architecture map. Later phases reference this file directly rather than re-discovering the file list. Combine with hard exclude lists for generated artifacts (`lib/src/rust/`, `rust/src/frb_generated.rs` here).

### C2 Trigger table for sync-driven mobile apps

**Why:** Mobile apps with multiple sync triggers (accessibility events, schedules, tiles, widgets, intents, deep links) need an explicit table because the trigger surface is the contract: each platform-channel-bound trigger is also a place where misconfiguration silently degrades the product.

**How to apply:** In the architecture map's Runtime Lifecycle, include a `Trigger | Owner | Path` table. Each entry names the source platform component, the event channel, and the destination Dart function. Downstream phases (contracts, protocols, defect-scan-semantic) cite this table rather than re-discovering each trigger's flow.

## Open Questions Left Behind

| ID | Kind | Description | Deferred Reason |
|---|---|---|---|
| arch-OQ3 | needs-maintainer-decision | Why is `lib/main.dart` 290KB single-file? Intentional bundling for tree-shaking, FRB single-registration-site requirement, or historical accumulation? | A maintainer-decision-level question; no pipeline phase can resolve it. Contracts phase still walks it page-by-page; this OQ is the meta-question. |

## Carry-Forward Routed

| ID | Target Phase | Description | Deferred Reason |
|---|---|---|---|
| arch-CF1 | contracts | Provider-API endpoint catalog (github_manager 70KB, gitlab_manager 47KB, gitea_manager 49KB, codeberg thin wrapper); pin retry/rate-limit behavior. | Contracts phase walks features. |
| arch-CF2 | contracts | AI feature contract (chat / wand / agent) across 11 files in `lib/api/ai_*.dart` plus SSE caching in `ai_stream_client.dart`. | Contracts phase rubric. |
| arch-CF3 | protocols | FRB wire surface: every Rust-side FFI function paired with its Dart caller. | Protocols phase rubric is exactly this. |
| arch-CF4 | protocols | flutter_background_service_with_intents (forked) intent broadcast protocol — accepted intents, payloads, reply broadcasts. Constants in `lib/gitsync_service.dart`. | Protocols phase. |
| arch-CF5 | protocols | home_widget IPC protocol (SharedPreferences keys, qualifiedAndroidName Receiver resolution, iOS WidgetCenter.reloadTimelines kind matching). | Protocols phase. |
| arch-CF6 | protocols | iOS BGTaskScheduler 101-ID pool semantics (slot assignment, behavior past container 100, register/unregister lifecycle). | Protocols phase. |
| arch-CF7 | defect-scan-mechanical | `lib/global.dart:demo` const flag (`TODO: Must be false for release`). | Pass 6 (config/environment). |
| arch-CF8 | defect-scan-mechanical | `mmap2_flutter` (forked) usage call-sites — concurrency hazards. | Mechanical pass enumerates dangerous patterns. |
| arch-CF9 | defect-scan-semantic | iOS-vs-Android `_finishWidget` Timer-vs-inline split — verify `_syncGeneration` race correctness. | Pass 3 (concurrency). |
| arch-CF10 | defect-scan-semantic | `secrets.dart` runtime use — verify no leakage to logs/errors/telemetry. | Pass 4 (security). |

## Framework Feedback (optional)

No significant friction observed in this phase. The pipeline structure (architecture before defect scan-mechanical before contracts/protocols) plus the carry_forward mechanic naturally absorbed the multi-language complexity of a Flutter+Rust app: structural observations went to the map, defect-flavored hunches got routed forward, and protocol-flavored hunches got routed forward. The architecture map template's split between primary (this map) and secondary outputs (`build-and-deploy`, `public-surfaces`, etc.) was useful for keeping the primary file scannable; downstream phases will append.

One small note: `analysis_options.yaml` ships `errors: constant_identifier_names: ignore`, which means linter won't flag `ALL_CAPS` constants in Dart code. The architecture map called this out implicitly via the constant names in `gitsync_service.dart` (`ACCESSIBILITY_EVENT`, `FORCE_SYNC`, ...) — useful style convention to flag if CONVENTIONS.md is ever populated for this project.

## Next Session Pointer

The next session is **defect-scan-mechanical** (passes 1, 2, 6 — logic/correctness, error handling, config/environment). Read in this order:

1. `.codecarto/GUIDE.md` and `.codecarto/workflow/status.yaml` (the latter has the next-actions list).
2. `.codecarto/findings/architecture/architecture-map.md` (full).
3. `.codecarto/scratch/file-inventory.md` (skim for size/role; pick deep-read targets).
4. `.codecarto/findings/defect-scan/passes/{01-logic-and-correctness,02-error-handling,06-config-and-environment}.md`.

Pick up `arch-CF7` (lib/global.dart:demo flag) and `arch-CF8` (mmap2_flutter call sites) from `phases.architecture.carry_forward` per status.yaml. Likely deep-read targets: `rust/src/api/git_manager.rs` (165KB), `lib/api/manager/git_manager.dart` (53KB), `lib/api/helper.dart` (29KB), `lib/api/logger.dart` (12KB), `lib/api/ai_tools_git.dart` (45KB), `lib/constant/values.dart`, `lib/constant/secrets.dart.template`, `pubspec.yaml`. Consider one subagent per pass file (passes 1/2/6) to keep primary context for finding classification and severity tagging. Output goes to `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md` per pipeline-full-with-deep-audit.yaml.
