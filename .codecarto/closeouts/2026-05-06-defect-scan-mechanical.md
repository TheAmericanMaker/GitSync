# Closeout — 2026-05-06 — defect-scan-mechanical

## Summary

Second implementing-session pass. Ran the mechanical subset of the defect-scan methodology (passes 1, 2, 6) over GitSync's 24 highest-value source files using four parallel general-purpose subagents and one primary-thread Pass 6 review. Produced `findings/defect-scan-mechanical/mechanical-defects.md` (~70KB, 182 findings: 1 critical, 32 high, ~75 medium, ~75 low) anchored on four canonical scratch reports (`scratch/defects-{rust-core,dart-sync,providers,ai}.md`). The single critical is R2.3 — the Rust core's `certificate_check` callback returns `CertificateCheckStatus::CertificateOk` unconditionally, disabling TLS/SSH cert verification across all libgit2 paths; combined with `set_verify_owner_validation(false)` and `safe.directory=*` at process init, three independent libgit2 trust checks are disabled. Routed 21 items to `defect-scan-semantic` (8 concurrency, 8 security, 5 contract-drift). `arch-CF7` (demo flag) resolved as `C6.1` (high). `arch-CF8` (mmap2 call sites) returned no hits in the scanned files — re-routed as `arch-CF8-residual` to Pass 3 with target lib/main.dart + UI page files. The contracts phase enters with the AI feature contract sketch and provider rate-limit summary already drafted, partly fulfilling `arch-CF1` and `arch-CF2`.

## Files Touched

- **Added:**
  - `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md`
  - `.codecarto/scratch/defects-rust-core.md` (subagent A, commit ce19de96)
  - `.codecarto/scratch/defects-dart-sync.md` (subagent B, commit ee619ecc)
  - `.codecarto/scratch/defects-providers.md` (subagent C, commit 4a1e8ba6)
  - `.codecarto/scratch/defects-ai.md` (subagent D, commit 9e41e5ce)
  - `.codecarto/closeouts/2026-05-06-defect-scan-mechanical.md` (this file)
- **Modified:**
  - `.codecarto/workflow/status.yaml` (defect-scan-mechanical -> complete; current_phase advanced to contracts; carry_forward populated with 21 entries routed to defect-scan-semantic)
  - `.codecarto/THREAD_LOG.md` (one new index line)
- **Deleted:** none

## Tests / Gates

| Gate | Result | Notes |
|---|---|---|
| Validation block per workflow/VALIDATE.md | PASS | All 6 mechanical-defects criteria PASS. |
| Output path matches pipeline `primary_output` | PASS | Matches `findings/defect-scan-mechanical/mechanical-defects.md` per pipeline-full-with-deep-audit.yaml. |
| Each finding has location, severity, evidence level, action | PASS | Every row in pass tables has all four. |
| Sub-pass coverage: at least 2 of 3 mechanical passes produced findings | PASS | All 3 (Pass 1: 87, Pass 2: 76, Pass 6: 18). |
| `dart analyze` / `cargo check` / `flutter test` | n/a | Source code is read-only in this analysis run. |

## Decisions Beyond Prompt

- **D004** | Spawn one defect-scan subagent per source area rather than one per pass. | Pass-per-subagent would have each subagent re-load every file once per pass; area-per-subagent loads each file once and scans against multiple passes simultaneously, cutting tool calls roughly 3×. Tradeoff: severity calibration drifts slightly between subagents, mitigated by primary-thread synthesis with a unified rubric.
- **D005** | Preserve subagent ID prefixes (R/D/P/A) verbatim in mechanical-defects.md rather than renumbering globally. | Back-traceability to scratch reports beats global ordering. A reader who wants the per-finding 1-3 line code excerpt can grep e.g. `R2.3` directly in the scratch file.
- **D006** | File P2.1 (provider httpX wrappers have no retry/rate-limit/401 handling) under Pass 2 not Pass 5. | Even though it's the same defect that contracts phase will catalog as a contract drift, mechanically it is an error-handling gap. The carry_forward to semantic captures the contract-drift framing via sem-cand-providers-3/-4.
- **D007** | Defer the 4-byte `arch-CF8-residual` mmap re-routing to Pass 3 (concurrency) rather than rerunning a focused mmap-grep subagent in this phase. | Pass 3's concurrency rubric is the right place to catch race conditions on memory-mapped reads anyway; rerunning the grep here would add latency for marginal benefit.

## Proposed Conventions

### C3 Area-per-subagent for multi-pass defect scans

**Why:** When a defect scan has multiple passes against the same files, spawning one subagent per pass each loading every file is N× slower than spawning one subagent per source area each running multiple passes. Severity-rubric drift between area-subagents is a real cost but is small relative to the load-time savings, and the primary thread's synthesis pass uses a unified rubric to normalise.

**How to apply:** In phases like defect-scan-mechanical (3 passes) or defect-scan-semantic (3 passes), partition the codebase by source area (e.g. "Rust core", "Dart sync", "providers", "AI subsystem") rather than by pass. Give each area-subagent the relevant pass-rubric files and ask for grouped findings tagged by pass ID. Synthesize in the primary thread.

### C4 Severity calibration anchors per project

**Why:** Severity grades (`critical`/`high`/`medium`/`low`) need project-specific anchors to be useful. Without them, subagents drift. For a sync app, `critical` should anchor to "data loss on push/pull" or "trust-boundary regression"; `high` to "silent sync drop" or "UI deadlock"; `medium` to "poor errors" or "observability gap"; `low` to "style with correctness implications."

**How to apply:** State the anchors explicitly in subagent prompts and in the primary report. Subagent reports calibrate by re-reading their prompt; primary synthesis can reclassify when an anchor is missed.

## Open Questions Left Behind

None for this phase. The architecture phase's `arch-OQ3` (why is main.dart 290KB) remains open as a maintainer-decision item; not closable here.

## Carry-Forward Routed

21 items — see `mechanical-defects.md` § "Routed To Semantic Phase" or `status.yaml:phases.defect-scan-mechanical.carry_forward`. Summary by classification:

| Classification | Count | Sample IDs |
|---|---|---|
| Concurrency (Pass 3) | 8 | defR-CF3, defR-CF6, defR-CF7, S-D-1, S-D-2, S-D-3, sem-cand-providers-1, arch-CF8-residual |
| Security (Pass 4) | 8 | defR-CF1, sem-cand-providers-2, A-SEM-1, A-SEM-2, A-SEM-3, A-SEM-4, A-SEM-5, A-SEM-6 |
| Contract drift (Pass 5) | 5 | defR-CF2, S-D-4, S-D-5, sem-cand-providers-3, sem-cand-providers-4 |

## Framework Feedback (optional)

- The pipeline split between mechanical and semantic defect passes is doing real work here. About 12% of mechanical findings would have been miscategorised as concurrency/security/contract issues if the rubric hadn't explicitly told the scanner to defer them — instead, they cluster cleanly in the carry_forward list ready for semantic-phase synthesis.
- The `defect-scan/passes/0[1-6]-*.md` files are reused verbatim by `defect-scan-mechanical` and `defect-scan-semantic` (passes 1+2+6 vs 3+4+5). This works but is implicit; the SKILL.md for the split phases relies on the implementer to know the split. Worked fine here but is worth documenting in the templates.
- The architecture map's carry_forward to mechanical was directly useful: `arch-CF7` (demo flag) became `C6.1`, and `arch-CF8` (mmap2) drove a deliberate grep-and-not-found result rather than the issue being silently missed.

## Next Session Pointer

The next session is **contracts**. Read in this order:

1. `.codecarto/GUIDE.md` and `.codecarto/workflow/status.yaml` (next-actions list).
2. `.codecarto/findings/architecture/architecture-map.md` (full).
3. `.codecarto/findings/defect-scan-mechanical/mechanical-defects.md` — specifically the "Rate-limit / retry summary" and "AI feature contract sketch" sections, which partially fulfil `arch-CF1` and `arch-CF2` and feed directly into the contracts behavioral table.
4. `.codecarto/findings/contracts/SKILL.md`.
5. `.codecarto/templates/behavioral-contracts.md`.
6. Spawn a subagent to walk `lib/main.dart` (290KB) section-by-section into `scratch/main-feature-inventory.md` per the approved plan. Each user-visible feature: trigger → defaults → outputs → side effects → persisted state → error behavior. The subagent should also enumerate `lib/ui/page/onboarding_setup.dart` (159KB).
7. Pick up `arch-CF1` and `arch-CF2` from `phases.architecture.carry_forward`.
8. Other deep-read targets: `lib/ui/dialog/manual_sync.dart` (84KB — the manual-sync orchestrator), `lib/ui/dialog/merge_conflict.dart` (61KB), `lib/ui/dialog/auth.dart` (22KB — OAuth flows), `lib/api/manager/auth/git_provider_manager.dart`. Provider clients are already inventoried in `scratch/defects-providers.md` — cite that for endpoint enumeration rather than re-walking.

Output goes to `.codecarto/findings/contracts/behavioral-contracts.md` per pipeline-full-with-deep-audit.yaml.
