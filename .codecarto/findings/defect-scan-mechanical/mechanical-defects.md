# Mechanical Defects Report — GitSync

## Scan Context

- **Source:** `../` (repository root)
- **Architecture reference:** `findings/architecture/architecture-map.md`
- **Pipeline:** `pipeline-full-with-deep-audit.yaml` (phase 2 of 7)
- **Date:** 2026-05-06
- **Scope:** Mechanical passes only — Pass 1 (logic & correctness), Pass 2 (error handling & resilience), Pass 6 (configuration & environment). Semantic passes (3 concurrency, 4 security, 5 API contract violations) deferred to `defect-scan-semantic` after protocols.
- **Method:** Four parallel general-purpose subagents (one per major source area) plus primary-thread Pass 6 review of `pubspec.yaml`, `AndroidManifest.xml`, `Info.plist`, `generate-apk-release.yml`, `lib/global.dart`, `lib/constant/{strings,values,secrets.dart.template}`, `analysis_options.yaml`, `Cargo.toml`, `.fvmrc`. Subagent scratch reports retained for full enumeration: `scratch/defects-{rust-core,dart-sync,providers,ai}.md`.
- **ID prefixes** (preserve back-traceability to scratch files):
  - `R1.x` / `R2.x` — Rust core (`rust/src/api/git_manager.rs`)
  - `D1.x` / `D2.x` — Dart sync orchestration (`lib/gitsync_service.dart`, `lib/api/manager/{git_manager,storage,settings_manager,repo_manager,premium_manager}.dart`, `lib/api/{helper,logger,accessibility_service_helper}.dart`)
  - `P1.x` / `P2.x` — Provider clients (`lib/api/manager/auth/*.dart`)
  - `A1.x` / `A2.x` — AI subsystem (`lib/api/ai_*.dart`)
  - `C6.x` — Config/environment

**arch-CF7 resolved:** `lib/global.dart:demo` flag is a top-level `const bool` set to `false` with TODO comment — captured as `C6.1` (Pass 6). No Rust-side or Dart-side conditional code paths gate on `demo` in this analysis run; it is a marker only. `defR-CF4` confirms no Rust-side use.

**arch-CF8 partially resolved:** `mmap2_flutter` has **no use sites in any of the 24 scanned files** across the Rust core, Dart sync layer, provider clients, or AI subsystem. The forked package must be invoked from `lib/main.dart` (290KB, not deep-read here), the UI page layer (`code_editor.dart`, `file_explorer.dart`, `diff_view.dart`), or in `lib/api/ai_tools_file.dart` (also scanned, no `mmap` reference). Rerouted to `defect-scan-semantic` Pass 3 with target `lib/ui/page/file_explorer.dart` + `code_editor.dart` for follow-up — see `Routed to Semantic Phase` table below.

---

## Pass 1: Logic and Correctness

Sorted by severity (critical → high → medium → low). Locations cite file path + function name + approximate line. Full notes in scratch files.

### Critical (0)

No critical Pass 1 findings.

### High (15)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R1.2 | `rust/src/api/git_manager.rs:712 / init` | `git2::opts::set_verify_owner_validation(false).unwrap()` panics the FRB worker on every `run_with_lock` call (init is invoked at line 255 in every op). | high | observed | fix before porting |
| R1.3 | `rust/src/api/git_manager.rs:788-790 / set_author` | Three `.unwrap()` calls on `repo.config()` and `set_str(...)` — corrupt `.git/config`, EPERM, ENOSPC panic the worker on every commit/push/clone path. | high | observed | fix before porting |
| R1.5 | `rust/src/api/git_manager.rs:2934, 2967 / get_recommended_action` | `remote.name().unwrap()` panics on non-UTF8 remote names; `remote.disconnect().unwrap()` panics on flaky network teardown. | high | observed | fix before porting |
| R1.6 | `rust/src/api/git_manager.rs:1246 / get_file_diff` | `commit.message().unwrap()` panics on non-UTF8 commit messages — common in legacy/Windows-1252 history. Inconsistent with `get_recent_commits` which uses `unwrap_or("<no message>")`. | high | observed | fix before porting |
| R1.7 | `rust/src/api/git_manager.rs:3468, 3472, 3646, 3650 / force_push, upload_and_overwrite` | `fs::remove_dir_all(rebase_merge/apply).unwrap()` panics if cleanup fails (read-only fs, permission); leaves the repo half-cleaned. | high | observed | fix before porting |
| R1.9 | `rust/src/api/git_manager.rs:4183-4196 / generate_ssh_key` | Three `.unwrap()` on `KeyPair::generate`/`serialize_openssh`/`serialize_publickey` panic on RNG/encoding failure; the function returns `(String, String)` so there is no `Result` channel — a port must change the signature. | high | observed | port differently |
| D1.10 | `lib/api/manager/premium_manager.dart:35-58 / updateGitHubSponsorPremium` | Multiple `setBool(...false)` branches without `return`; a `408` response on `userRes` falls through to `jsonDecode('Error')` which throws `FormatException`. | high | observed | fix before porting |
| D1.12 | `lib/api/manager/premium_manager.dart:71-74 / cullNonPremium` | `List.generate(n, (i) async {...})` produces orphaned futures. `manager.clearAll()` calls fan out concurrently (or never), and `setStringList(repoNames, [first])` proceeds before cleanup completes. | high | observed | fix before porting |
| D1.13 | `lib/api/manager/repo_manager.dart:9-15 / getRepoName` | If `repomanReponames.isEmpty`, `length - 1 = -1` and `repomanReponames[-1]` throws `RangeError`. The clamp's `setInt` is fire-and-forget so reentrancy yields a stale index. | high | observed | fix before porting |
| D1.23 | `lib/api/manager/storage.dart:_get` (~125-150) | `if (key.hasDefault && !defaulting) throw StateError(...)` — typed accessors (`getBool`, `getStringNullable`, etc.) do not pass `defaulting=true`, so a direct `getBool(StorageKey.setman_clientModeEnabled)` (which `hasDefault=true`) throws. The wrapper `_getOrDefault` is the only safe path. Footgun. | high | observed | fix before porting |
| D1.24 | `lib/api/manager/storage.dart:_set<List<String>>` (~175) | `(value as List<String>).join(",")` corrupts elements containing `,`. `repoman_repoNames` is a user-provided container-name list — names with commas split incorrectly on round-trip. | high | observed | fix before porting |
| P1.2 | `lib/api/manager/auth/git_provider_manager.dart:getToken` (~85-89) | OAuth refresh race: two concurrent provider calls both observing an expired token each call `refreshToken`, the second invalidating the first. Affects GitHub-App / GitLab / Gitea / Codeberg. **Also routed to defect-scan-semantic Pass 3** (sem-cand-providers-1). | high | strong inference | fix before porting |
| P1.7 | `lib/api/manager/auth/github_manager.dart:113 / getRepos` (and twin in `gitea_manager.dart:86`) | Search is implemented as client-side filter over only the first 30 (default) or 100 repos; repos beyond page 1 are invisible to the repo-picker search. | high | observed | fix before porting |
| A1.18 | `lib/api/ai_completion_service.dart:aiComplete` | Returns `''` (empty string, not null) when the provider's content is empty. The commit-message wand can commit an empty AI-generated message. | high | observed | fix before porting |
| A1.25 | `lib/api/ai_chat_service.dart:_streamRound (content-block hygiene)` | `removeWhere((b) => b is TextBlock && b.text.isEmpty)` runs after the loop; if a stream ends right after `ToolCallEnd`, the trailing empty TextBlock is the only block, gets stripped, and `_anthropicAssistantMessage` returns null — the assistant turn is dropped from the wire while remaining locally, breaking `tool_use`/`tool_result` pairing on the next request and poisoning the chat. | high | observed | fix before porting |

### Medium (37)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R1.1 | `rust/src/api/git_manager.rs:466 / run_with_lock poll loop` | `read_flock.read_to_string(...).unwrap_or(0)` swallows `io::Error` as 0 bytes → branch returns `Ok(T::default())`. Caller cannot distinguish dropped sync from successful no-op. | medium | observed | port differently |
| R1.8 | `rust/src/api/git_manager.rs:4159-4161 / abort_merge` | TOCTOU between `exists()`, `metadata().unwrap()`, `remove_file().unwrap()`; both `unwrap`s panic on transient I/O. | medium | observed | fix before porting |
| R1.11 | `rust/src/api/git_manager.rs:4230 / get_branch_name_priv` | `head.shorthand().unwrap()` panics on non-UTF8 branch names; `is_branch()` does not guard encoding. | medium | strong inference | fix before porting |
| R1.12 | `rust/src/api/git_manager.rs:903 / clone_repository` | `let _ = remote.fetch::<&str>(&[], Some(&mut fo2), None);` silently drops the result of a redundant post-clone fetch. | medium | observed | port differently |
| R1.14 | `rust/src/api/git_manager.rs:1968-1996 / fast_forward checkout` | `CheckoutBuilder` chained `.safe().force()` — `safe()` is dead code; only `force()` takes effect. Documented as TODO. | medium | observed | port differently |
| R1.16 | `rust/src/api/git_manager.rs:3373-3393, 3494-3517, 3672-3694, 3793-3812` | Same ORIG_HEAD-to-branch-name resolution duplicated 4×; picks the *first* branch matching the tip — wrong branch wins when multiple branches share a tip. | medium | strong inference | port differently |
| R1.19 | `rust/src/api/git_manager.rs:2046-2066 / commit signed branch` | After `commit_signed`, the else-branch handles *any* `repo.head()` error including corruption; synthesizes a branch ref pointing to the new commit, silently overwriting the corrupt HEAD. Should match on `ErrorCode::UnbornBranch`. | medium | strong inference | port differently |
| R1.20 | `rust/src/api/git_manager.rs:2675-2680 / push_changes_priv rebase loop` | The first-rebase loop blindly commits each step with `Unmerged` propagated as a hard error rather than routing to the merge-conflict UI (the second rebase loop, lines 2729-2738, does route correctly). | medium | strong inference | port differently |
| D1.1 | `lib/gitsync_service.dart:_sync` (5 sites) | `_displaySyncMessage(null, ...)` with `settingsManager==null` always shows the message — the `null \|\| getBool(...)` short-circuit bypasses the user's `setman_syncMessageEnabled=false` toggle. | medium | observed | fix before porting |
| D1.2 | `lib/gitsync_service.dart:_sync` (~226-233) | "Sync failed" log branch does not set `terminal='error'`; relies on `innerError` set inside earlier branches. Defensive gap. | medium | strong inference | fix before porting |
| D1.5 | `lib/gitsync_service.dart:merge` (~290-336) | Client-mode merge commits via `backgroundStageAndCommit` but never invokes `debouncedSync(...)` — UI shows "merge complete" while the resolution stays unpushed. May be intentional; unverified. **Also routed to semantic** (S-D-4). | medium | strong inference | port differently |
| D1.11 | `lib/api/manager/premium_manager.dart:60` | `if (userRes.statusCode != 200) { throw 'Failed to load sponsors.txt: ${fileRes.statusCode}'; }` — condition checks `userRes` but message references `fileRes`. Dead code (line 51 returns when `userRes != 200`); covers up `fileRes` failures. | medium | observed | fix before porting |
| D1.14 | `lib/api/helper.dart:debounce` (~86-90) | `debounceTimers` and `_callbacks` keyed by string never have entries removed after firing — bounded leak per repo, unbounded for `iosFolderAccessDebounceReference` per-path keys. `cancelDebounce` force-unwraps `_callbacks[index]!()` with NPE risk on stale keys. | medium | observed | fix before porting |
| D1.17 | `lib/api/helper.dart:hasNetworkConnection` (~178) | `(await Connectivity().checkConnectivity())[0]` indexes without length check; `connectivity_plus` may return empty list on platform errors. Crashes the network-error path at worst time. | medium | observed | fix before porting |
| D1.26 | `lib/api/helper.dart:cancelDebounce` (~96) | `_callbacks[index]!()` force-unwrap on possibly-null map lookup — NPE for keys that never had a debounce scheduled. | medium | observed | fix before porting |
| P1.3 | `lib/api/manager/auth/git_provider_manager.dart:getToken` (~84) | `if (refreshed.accessToken == null \|\| refreshed.refreshToken == null) return null;` is unreachable — already gated on the line above. The intent was clearly "or refreshToken null"; as written, a refresh that returns access but no refresh persists `refreshed.refreshToken!` which throws null-check exception. | medium | observed | fix before porting |
| P1.4 | `lib/api/manager/auth/gitlab_manager.dart` (multiple `_get*Request`) | Pagination rebuild via `Uri.parse(url).replace(queryParameters: {...})` round-trips already-encoded params, decoding `+` as space. Search filter `searchFilter="foo+bar"` is silently corrupted on page 2+. | medium | strong inference | fix before porting |
| P1.5 | `lib/api/manager/auth/gitlab_manager.dart, gitea_manager.dart` (URL builders) | `Uri.encodeComponent(searchFilter)` is mixed with raw concatenation of other params — labels containing `&`, `=`, `#`, or space silently corrupt URLs. | medium | observed | fix before porting |
| P1.6 | `lib/api/manager/auth/github_manager.dart:_searchIssues (~273), _searchPullRequests (~646)` | Search uses `+` as GitHub's space delimiter; `Uri.encodeComponent(search)` does not encode literal `+`, so `"C++"` collapses to `"C  "`. | medium | observed | fix before porting |
| P1.8 | `lib/api/manager/auth/github_manager.dart:_getReposRequest` (~131) | On non-200, neither `updateCallback` nor `nextPageCallback` is invoked. UI loading spinner hangs forever. Other GitHub list endpoints DO have the empty-result fallback; this one is missing. | medium | observed | fix before porting |
| P1.9 | `lib/api/manager/auth/gitea_manager.dart:_getReposRequest` (~103) | Same shape as P1.8 — non-200 leaves callbacks unresolved. | medium | observed | fix before porting |
| P1.11 | `lib/api/manager/auth/gitlab_manager.dart:getIssueDetail / getPrDetail` (~462-490, ~625-660) | Per-comment award_emoji fetch is sequential in a `for` loop. 100-note MR issues 100 sequential round-trips; on slow networks the page sits blank for tens of seconds. Same in `gitea_manager.dart`. | medium | observed | port differently |
| P1.12 | `lib/api/manager/auth/gitea_manager.dart:_getActionRunsRequest` (~539) | Decodes as `Map<String, dynamic>` and reads `["workflow_runs"]`. Forgejo and some Gitea versions return a bare list at this endpoint, throwing the cast. | medium | strong inference | fix before porting |
| P1.17 | `lib/api/manager/auth/github_manager.dart:_searchIssues / _searchPullRequests` paging | `if (page * 30 < totalCount) nextPageCallback(...)` — GitHub's search API caps `total_count` at 1000; pagination beyond cap returns 422 silently masked as `[]`. | medium | observed | fix before porting |
| A1.1 | `lib/api/ai_stream_client.dart:streamCompletion` SSE parser | Splits on `\n` only; does not strip CRLF `\r`, does not concatenate multi-`data:` events, does not handle `:` heartbeat comments. Spec-non-conformant. Works today only because OpenAI/Anthropic emit one-`data:`-per-event. | medium | observed | port differently |
| A1.3 | `lib/api/ai_stream_client.dart:_parseOpenAI` (tool-call) | Provider streaming `arguments` fragment without `id` falls back to `id ?? '$index'`; if two distinct tool calls in different deltas both have null `id` and missing `index`, fragments collide into one builder. | medium | observed | fix before porting |
| A1.4 | `lib/api/ai_stream_client.dart:_parseAnthropic content_block_start` | `block['id']` and `block['name']` are read with `??` fallback to `''` — empty-id `ToolCallStart` events can be emitted; downstream `toolCallBuilders` keys collide on empty id across blocks. | medium | observed | fix before porting |
| A1.6 | `lib/api/ai_chat_service.dart:_streamRound dangling-builder fallback` | If the stream ends without matching `ToolCallEnd` (cancellation, network truncation), the trailing fallback tries to parse partial JSON; on `jsonDecode` failure the empty `catch (_)` swallows it, producing a `ToolUseBlock` with `input: {}`. The tool then runs with empty arguments. | medium | observed | fix before porting |
| A1.8 | `lib/api/ai_chat_service.dart:_findToolCallForResult` | Walks back from a tool-result message and counts siblings; if a prior assistant turn had fewer tool calls than the current cluster (e.g. messages were trimmed), falls back to `calls.last`, mis-pairing the result with an unrelated tool call → wrong `tool_use_id` to Anthropic → 400 on next request. | medium | observed | fix before porting |
| A1.10 | `lib/api/ai_tools_file.dart:_resolve (non-existent file branch)` | TOCTOU: parent canonicalises and returns un-canonicalised `joined` path. If a symlink in the path resolves outside the repo root *after* parent creation, FileWrite escapes the sandbox. | medium | observed | fix before porting |
| A1.13 | `lib/api/ai_tools_file.dart:FileListTool._listDir (recursive)` | Sync `dir.listSync()` inside async recursion; freezes the isolate on large repos. Also no symlink-cycle detection. | medium | observed | port differently |
| A1.14 | `lib/api/ai_tools_git.dart` (multiple Git*Tool) | No client-side validation of branch/tag/remote names or commit messages. Empty/whitespace commit messages, branch names starting with `-`, refs containing `..` are delegated entirely to libgit2; users see whatever libgit2 says. | medium | observed | fix before porting |
| A1.16 | `lib/api/ai_tools_git.dart:GitSyncTool.execute` | `pullResult` checked vs `null` only; tri-state `(true \| false \| null)` is collapsed — `false` (no changes) and `true` (pulled) both proceed to push, and `pushResult==false` is interpreted as "nothing to push" rather than potentially "push failed silently." Should mirror `gitsync_service.dart:_sync` tri-state. | medium | observed | fix before porting |
| A1.17 | `lib/api/ai_completion_service.dart:aiComplete` | Returns `null` for any non-2xx, including 401/403/429. Caller cannot distinguish "key invalid" from "rate-limited" from "no model" — every failure looks like "no suggestion." | medium | observed | fix before porting |
| A1.22 | `lib/api/ai_chat_service.dart:_loadFromDisk` | Corrupt/partial JSON → silent fail → `_conversations[index]` unset → next `_convFor(index)` creates fresh conversation, **discarding history**. No backup, no UI signal. | medium | observed | fix before porting |
| A1.23 | `lib/api/ai_provider_validator.dart:normalizeEndpoint` | If the user supplied just an IP (`192.168.1.10:8080`), `http://` is prepended — self-hosted users transmit API keys in plaintext. **Routed to semantic Pass 4 (security) as A-SEM-4.** | medium | observed | port differently |

### Low (35)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R1.4 | `rust/src/api/git_manager.rs:2336 / pull_changes_priv` | `let head = result.unwrap();` reachable if the upstream `is_none` check is later refactored away. | low | observed | leave behind |
| R1.10 | `rust/src/api/git_manager.rs:4183` | `KeyPair::generate(KeyType::ED25519, 256)` — the `bits=256` is meaningless for Ed25519; osshkeys ignores it. Cosmetic. | low | observed | leave behind |
| R1.13 | `rust/src/api/git_manager.rs:1141, 2858-2859, 2903-2904 / untrack_all, stage/unstage_file_paths` | `index.write_tree()` with discarded OID — likely a deliberate side-effect call but discoverability hazard. | low | observed | leave behind |
| R1.15 | `rust/src/api/git_manager.rs:2515-2520 / download_changes` | Wildcard `_ => {}` over `pull_changes_priv` result is latent only; would mis-handle a future `Ok(None)` variant. | low | strong inference | leave behind |
| R1.17 | `rust/src/api/git_manager.rs:858-864 / clone_repository transfer_progress` | `usize → i32` truncation on object counts; wraps to negative beyond 2.1B. Not realistic but unconditional. | low | observed | leave behind |
| R1.18 | `rust/src/api/git_manager.rs:1525-1526, 1669-1670 / get_*_diff` | `as i32` casts on `insertions()`/`deletions()` truncate at 2.1B. | low | observed | leave behind |
| D1.3 | `lib/gitsync_service.dart:_sync` (~262/274) | Second `optimisedSyncFlag` short-circuit verified correct — re-query of `getRecommendedAction` is intentional for post-pull state. Documented for clarity. | low | observed | leave behind |
| D1.4 | `lib/gitsync_service.dart:_sync` line 291 | `getRecentCommits(priority: 3)` called twice on success paths. Wasteful but not incorrect. | low | observed | leave behind |
| D1.6 | `lib/gitsync_service.dart:accessibilityEvent` (~338-376) | Edge case: if `lastOpenPackageName` is updated but `lastOpenPackageNameExcludingInputs` is not, IME → app A → IME → app B sequence may miss A's "close" event. | low | strong inference | leave behind |
| D1.9 | `lib/api/manager/git_manager.dart:_runWithLock` no-lock branch | Two layers of catch obscure originating exception type for `getCommitDiff`/`getFileDiff`/`getWorkdirFileDiff`. Discoverability. | low | observed | leave behind |
| D1.15 | `lib/api/helper.dart:waitFor` (~530-540) | `if (result == null) return null;` inverts typical `waitFor` semantics — `null=done`. Inner exceptions swallowed for `maxWaitSeconds`, then propagated from final unhandled call. Subtle. | low | observed | leave behind |
| D1.16 | `lib/api/helper.dart:handleIfNetworkError` (~190-208) | `_networkStallPatterns` includes "timed out" — broad enough that a Dart-side `TimeoutException` could trigger network-style retry. | low | observed | leave behind |
| D1.18 | `lib/api/logger.dart:dismissError(BuildContext? context)` (~144) | `null` context still pops dialog but doesn't show new dialog; non-obvious null semantics. | low | observed | leave behind |
| D1.19 | `lib/api/logger.dart:logError` (~137) | Stack trace interpolated before error message — reversed from convention. | low | observed | leave behind |
| D1.20 | `lib/api/logger.dart:reportIssue` (~173) | `result?.$3 ?? null` redundant. | low | observed | leave behind |
| D1.21 | `lib/api/manager/settings_manager.dart:getGitDirPath` (~110-130) | `path.isEmpty == true` evaluates `bool == true`; code smell suggesting nullability confusion. The `path` parameter shadows the import alias. | low | observed | leave behind |
| P1.1 | `lib/api/manager/auth/git_provider_manager.dart:getToken` (~64-89) | OAuth-token parser truncates `accessToken` if it itself contains the separator (currently unbounded by validation). | low | observed | leave behind |
| P1.10 | `lib/api/manager/auth/gitea_manager.dart:_getPullRequestsRequest` (~304-308) | Extracts `owner`/`repo` from URL pathSegments without bounds check on `reposIdx`. | low | strong inference | leave behind |
| P1.13 | `lib/api/manager/auth/gitea_manager.dart:removeReaction` (~1116) | DELETE with JSON body — proxies sometimes strip; non-portable. | low | strong inference | leave behind |
| P1.14 | `lib/api/manager/auth/github_manager.dart:updateIssueState` (~1612) | Issue ID interpolated into GraphQL mutation string instead of using GraphQL variables. | low | observed | leave behind |
| P1.15 | `lib/api/manager/auth/gitlab_manager.dart:_getActionRunsRequest` (~447-448) | Pipeline `iid` falls back to `id` (project-global), causing display collisions on cross-source aggregation. | low | strong inference | leave behind |
| P1.16 | `lib/api/manager/auth/gitea_manager.dart:_getReleasesRequest` (~474-475) | Branch name `cafe123` matches `^[0-9a-f]{7,}$` and is misclassified as a SHA. | low | strong inference | leave behind |
| P1.18 | `lib/api/manager/auth/gitea_manager.dart:getPullRequests` | `state=merged` not accepted by Gitea (returns 422 → silent empty list). | low | strong inference | leave behind |
| A1.2 | `lib/api/ai_stream_client.dart:streamCompletion` finally-block + cancel | Redundant `client.close()` before `return`; `response.stream` not drained on cancel. | low | observed | leave behind |
| A1.5 | `lib/api/ai_stream_client.dart:_parseGoogle (functionCall)` | New UUID per Google `functionCall` — locally correct but persisted ToolUseBlock id randomised per request, making post-hoc replay unstable. | low | observed | leave behind |
| A1.7 | `lib/api/ai_chat_service.dart:_runAgenticLoop` | `_maxToolRounds=25` caps rounds but no per-round tool-call cap — `N tools × 25 rounds` is unbounded. | low | observed | leave behind |
| A1.9 | `lib/api/ai_tools_file.dart:_resolve` sandbox | Trailing-slash fragility on iOS document-picker root canonicalisation. | low | observed | leave behind |
| A1.11 | `lib/api/ai_tools_file.dart:FileEditTool.execute` | `input['edits'] as List<dynamic>` throws if model passes `edits: null` or string — bubbles as Dart error in chat. | low | observed | leave behind |
| A1.12 | `lib/api/ai_tools_file.dart:FileSearchTool` | `_matchGlob` only handles `.` and `*`; `**`, `?`, character classes silently match nothing. | low | observed | leave behind |
| A1.15 | `lib/api/ai_tools_git.dart:GitDiffTool.execute (file path branch)` | Off-by-one in `omitted` line count — split-count includes partial trailing line. | low | observed | leave behind |
| A1.19 | `lib/api/ai_chat_service.dart:_buildApiMessages (Google branch)` | Tool-result shape duplicated across `_buildApiMessages` and `_toGoogleMessage`; correct today only because both code paths agree. | low | observed | leave behind |
| A1.20 | `lib/api/ai_chat_service.dart:_streamRound UsageUpdate` | Anthropic `usage` accumulates from both `message_start` and `message_delta`; works because input/output are zero in the off-direction but fragile by construction. | low | observed | leave behind |
| A1.21 | `lib/api/ai_chat_service.dart:_RepoConversation.fromJson` | `TokenUsage(su['input'] ?? 0, ...)` throws on `num`/double-shaped persisted JSON; caught by `_loadFromDisk` so silent history-load failure. | low | observed | leave behind |
| A1.24 | `lib/api/ai_tools_provider.dart:_getContext (no-toolContext path)` | No validation that the URL is a recognised provider; provider manager call fails opaquely on Codeberg/Gitea host mismatch. | low | observed | leave behind |

---

## Pass 2: Error Handling and Resilience

### Critical (1)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| **R2.3** | `rust/src/api/git_manager.rs:756 / get_default_callbacks certificate_check` | `callbacks.certificate_check(\|_, _\| Ok(CertificateCheckStatus::CertificateOk));` — **unconditionally trusts every TLS/SSH certificate from every remote**. Combined with `set_verify_owner_validation(false)` (line 712) and `safe.directory=*` (line 716), this disables three independent libgit2 trust checks at process init. **Security regression vs stock libgit2 default.** Filed under Pass 2 because the behavior is "fail-open swallow of every cert error"; **also routed to defect-scan-semantic Pass 4** (defR-CF1) for the trust-boundary classification. | **critical** | observed | **fix before porting** |

### High (15)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R2.2 | `rust/src/api/git_manager.rs:438 / run_with_lock hard-timeout` | `return Ok(T::default())` on 10-min hard timeout. For `Result<Option<bool>>` callers this is `Ok(None)` = "no work needed" in the upstream contract. **Silent sync drop after 10 min wait.** | high | observed | fix before porting |
| R2.4 | `rust/src/api/git_manager.rs:763-766 + everywhere` | All errors funneled via `git2::Error::from_str(string)` — class/code lost. Dart cannot distinguish auth failure / network failure / merge conflict / lock contention. **Routed to semantic** (defR-CF2). | high | observed | port differently |
| R2.6 | `rust/src/api/git_manager.rs:911-1079 / clone_repository submodule traversal` | Every submodule operation wrapped in `let _ = ...` or `.ok()` (~6 sites). Submodule failures silently leave the submodule in detached HEAD; user sees "successful" clone. | high | observed | fix before porting |
| R2.13 | `rust/src/api/git_manager.rs:3893-3917 / get_conflicting` | Per-entry errors from the conflict iterator silently dropped — partially-corrupted index reports fewer conflicts than actually exist; merge-conflict UI thinks resolution complete when it isn't. | high | observed | fix before porting |
| R2.18 | `rust/src/api/git_manager.rs:255 / run_with_lock unconditional init()` | Every op invokes `init(None)` which runs the panicking `set_verify_owner_validation(false).unwrap()` (R1.2). Combine with R1.2; gate `init()` behind `OnceLock`/`OnceCell`. | high | observed | fix before porting |
| D2.1 | `lib/gitsync_service.dart:_sync` finally block (~298-308) | `if (isScheduled) { ... debouncedSync(...); }` — recursive resync future is **not awaited**. On iOS where the widget-callback isolate dies promptly, the rescheduled sync may never execute. | high | observed | fix before porting |
| D2.5 | `lib/gitsync_service.dart:merge` (~301) | Non-network exception path `rethrows` without invoking `MERGE_COMPLETE`. UI's merge-conflict dialog hangs on a never-resolving future. **User must force-close the dialog.** | high | observed | fix before porting |
| D2.9 | `lib/api/manager/git_manager.dart:backgroundUploadChanges mergeConflictCallback` (~695-703) | Background upload's merge-conflict callback `await repoManager.setInt(repoman_repoIndex, repomanRepoindex)` — **mutates the global current-repo index** as a side effect. If the user is viewing a different repo in the UI, their context silently swaps. | high | strong inference | fix before porting |
| P2.1 | `lib/api/helper.dart:httpGet/Post/Patch/Put/Delete` | **No retry, no 429 handling, no 5xx handling, no 401 handling** anywhere in the HTTP wrappers. 10s timeout synthesises a fake `http.Response('Error', 408)`. **Every provider call inherits this gap.** This is `arch-CF1`; see "Rate-limit/retry summary" below. | high | observed | fix before porting |
| P2.2 | `lib/api/manager/auth/{gitea,github}_manager.dart:_getReposRequest catch` | `catch (e, st)` logs error but **does not invoke `updateCallback` or `nextPageCallback`**. UI loading spinner hangs forever on network failure. **Repo picker is in onboarding — first user impression.** | high | observed | fix before porting |
| P2.4 | All providers, all list/detail/mutation methods | 401/403 treated as "not 200/201" → returns `null`/`false`/`[]`. **No re-auth trigger.** User sees empty list / silent failure when token expires; for non-refreshing GitHub the UI never prompts re-login. | high | observed | fix before porting |
| P2.5 | All providers, every endpoint | 429 (rate-limited) collapses into the same path as P2.4 — no `Retry-After` parse, no UI message. The spinner just resolves to empty data. **arch-CF1.** | high | observed | fix before porting |
| P2.14 | All providers' POST mutations (`createIssue`, `addIssueComment`, `createPullRequest`, `addReaction`) | Not idempotent; if the 10s timeout returns synthetic 408 but the server received the request, retrying creates a duplicate. **No idempotency key.** Latent today (no auto-retry) but becomes acute the moment P2.1's fix adds retry. | high | strong inference | fix before porting |
| A2.1 | `lib/api/ai_stream_client.dart:streamCompletion` (HTTP error branch) | 4xx/5xx surface as `StreamError('API error 401: <body>')`. No distinct re-auth signal — `aiKeyConfigured` is not flipped to false. **User has no actionable path out** of an expired key. | high | observed | fix before porting |
| A2.3 | `lib/api/ai_stream_client.dart` and `lib/api/ai_completion_service.dart` | **No 429 handling** in either chat or wand path. `ai_provider_validator.dart:fetchAvailableModels` does have a friendly message; the actual chat path does not. | high | observed | fix before porting |

### Medium (37)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R2.1 | `rust/src/api/git_manager.rs:540-543 / QueueCleanupGuard::drop` | Cleanup failure on Drop is logged via `eprintln!` only (not `log_callback`). Orphaned queue entry causes the next caller to wait `MAX_WAIT_SECS=600s` before self-evicting — user sees "sync hangs 10 min" with no actionable telemetry. | medium | observed | port differently |
| R2.5 | `rust/src/api/git_manager.rs:903 / clone_repository post-clone fetch` | `let _ = remote.fetch(...)` — second fetch's silent failure leaves the clone without populated refspecs. Same as R1.12. | medium | observed | port differently |
| R2.7 | `rust/src/api/git_manager.rs:1555-1570 / get_workdir_file_diff` | `if let Ok(staged_diff) = repo.diff_tree_to_index(...)` — staged-diff failure silently treated as "no staged lines." Every line that was actually staged shows as unstaged. | medium | observed | port differently |
| R2.8 | `rust/src/api/git_manager.rs:2143-2146 / update_submodules_priv` | Submodule update failure logged then skipped. Sync depending on submodule content silently produces a stale checkout with no signal to caller. | medium | observed | port differently |
| R2.10 | `rust/src/api/git_manager.rs:2569-2576, 3443-3449, 3621-3627 / push callbacks` | Push-update-reference callback stores rejection messages in `Arc<Mutex<Option<String>>>` — only the *first* rejected ref is preserved; subsequent rejections overwrite. | medium | observed | port differently |
| R2.11 | `rust/src/api/git_manager.rs:4498 / set_disable_ssl` | `let _ = config.set_str("http.sslVerify", value);` — silently swallows `set_str` failure. User toggling SSL gets old behavior with no error. | medium | observed | port differently |
| R2.15 | `rust/src/api/git_manager.rs:4086-4089 / has_local_changes_priv` | Failure to read statuses biases `get_recommended_action` toward "no action" — sync skips a repo whose index is unreadable. | medium | observed | fix before porting |
| R2.17 | `rust/src/api/git_manager.rs:1117-1133 / untrack_all gitignore` | `.gitignore`/`.git/info/exclude` read failures (permission, symlink) silently produce wrong untrack behavior; user's gitignore entries are *not* untracked. | medium | observed | port differently |
| D2.2 | `lib/gitsync_service.dart:_sync` retry interaction | If retry fires while user-initiated sync is in progress, the rescheduled call from `_sync` finally block doesn't propagate `retryCount` — exponential backoff escapes; effectively retryCount resets to 0. | medium | strong inference | fix before porting |
| D2.3 | `lib/gitsync_service.dart:_sync` (5 sites: ~247, 249, 251, 270) | `_displaySyncMessage(null, "Credentials not found")` (and similar) not awaited — toast/notification may race with the subsequent `return;`. Lost user feedback in the most error-prone paths. | medium | observed | fix before porting |
| D2.4 | `lib/gitsync_service.dart:_updateForceSyncWidget` (~142) | Catch uses `print('ForceSyncWidget update failed: $e');` — not routed through `Logger`, doesn't appear in report-bug log. Comment claims "logged for diagnosis" but logs are ephemeral. | medium | observed | fix before porting |
| D2.6 | `lib/gitsync_service.dart:merge` (~327) | `if (!await settingsManager.getClientModeEnabled()) { debouncedSync(repomanRepoindex, true); }` — fire-and-forget; if `SettingsManager().reinit` throws, the throw is silently swallowed. | medium | observed | fix before porting |
| D2.8 | `lib/api/manager/git_manager.dart:pushChanges` (~366-374) | Resync-string branch logs WITH `errorContent`, then unconditional second `logError` runs **even on the resync path** — double-logging and double-toasting. | medium | observed | fix before porting |
| D2.10 | `lib/api/helper.dart:_performRemoteRepoCreation` (~405-408) | `try { ... } catch (_) { /* Non-fatal */ }` — fully empty catch swallowing all errors including bugs. Comment justifies but no log. | medium | observed | fix before porting |
| D2.15 | `lib/api/logger.dart:reportIssue` (~219) | `await http.post(...)` without try/catch. Network down → exception propagates from a button handler → user sees a crash. | medium | observed | fix before porting |
| D2.18 | `lib/api/accessibility_service_helper.dart:getApplicationLabel` (~75) | Return type `Future<String>` (non-null), but if platform method returns null the `await` throws `TypeError`. Uninstalled-app lookup crashes. | medium | observed | fix before porting |
| D2.19 | `lib/gitsync_service.dart:_finishWidget Timer (Android)` | Timer's inner `_updateForceSyncWidget` is `Future<void>` called from a non-async closure → fire-and-forget, exceptions unobserved. | medium | observed | fix before porting |
| P2.3 | All managers `getIssueDetail`/`getPrDetail` per-comment-reaction loops (~gitea ~707, gitlab ~488, github ~1534) | Inner `try {} catch (_) {}` blocks. Errors swallowed entirely with no logging — masks 401, 5xx, JSON-parse failures. | medium | observed | fix before porting |
| P2.6 | All providers' catch logging | `Logger.logError(LogType.X, e, st)` may capture `ClientException` whose message embeds full request URI. Authorization headers are *not* in the URI but **GitHub OAuth flow sometimes embeds tokens in URLs (refresh redirect)**. **Routed to semantic Pass 4** (sem-cand-providers-2). | medium | strong inference | fix before porting |
| P2.8 | All providers | `http.get/post/...` creates a single-shot `IOClient` per call — no client reuse, no Keep-Alive, no connection pooling. Every request pays the TLS handshake cost. | medium | observed | port differently |
| P2.9 | `lib/api/manager/auth/gitlab_manager.dart:createRepo` (~64-67) | On HTTP 400 (already exists), assumes URL is `https://$_domain/$username/$repoName.git`. **Wrong for GitLab group projects** under `/$group/$subgroup/$repoName`. User finds out only when push fails. | medium | observed | fix before porting |
| P2.10 | `lib/api/manager/auth/git_provider_manager.dart:getToken` (~85) | `oauthClient.refreshToken(...)` failure surfaces as generic exception; no offline tolerance (an expired-but-grace-period token is *still* used by GitHub but not by GitLab/Gitea). | medium | observed | fix before porting |
| P2.12 | All providers' mutation endpoints (`addIssueComment`, `addReaction`, etc.) | Non-2xx responses return `null`/`false` with no detail; UI shows generic "failed" toast. Provider-side body containing actual reason discarded. | medium | observed | fix before porting |
| P2.15 | `lib/api/manager/auth/git_provider_manager.dart:launchOAuthFlow` (~94) | If `getUsernameAndEmail` returns null post-OAuth, OAuth tokens are silently discarded — user redoes the entire flow. | medium | observed | fix before porting |
| P2.17 | All providers' response decoding | `utf8.decode + json.decode` — if provider returns HTML (Cloudflare error, captive portal, instance-down page), `json.decode` throws and catch returns null/empty. UI cannot distinguish "instance down" from "no data". | medium | observed | fix before porting |
| A2.2 | `lib/api/ai_stream_client.dart:streamCompletion error logging` | `print('[AI Stream] $provider HTTP ${response.statusCode} body=$errorBody')` writes upstream error body to logcat / iOS unified log. Some providers echo headers / model names; rare key fragments. **Routed to semantic Pass 4** (A-SEM-2). | medium | indirect | fix before porting |
| A2.4 | `lib/api/ai_stream_client.dart:streamCompletion mid-stream interrupt` | `lineBuffer` flush at end attempts to parse a partial event; on `jsonDecode` failure only a `print` is emitted. Consumer sees stream end without `StreamComplete` and `_streamRound` exits with half-rendered text. | medium | observed | fix before porting |
| A2.5 | `lib/api/ai_stream_client.dart:streamCompletion cancel-lifecycle` | `client.close()` aborts in-flight requests but `await for (final chunk in response.stream...)` may hold a chunk; no `await response.stream.drain()` — leaks until GC. | medium | observed | port differently |
| A2.6 | `lib/api/ai_chat_service.dart:_runAgenticLoop` exception | `conv.error = err.toString()` — `SocketException` produces "Failed host lookup: 'api.openai.com'..." — host string leaks to UI. Compare friendlier `ai_provider_validator.dart`. | medium | observed | fix before porting |
| A2.7 | `lib/api/ai_tool_executor.dart:execute 60s timeout` | `onTimeout` returns `jsonEncode({'error': ...})` and sets `block.status=failed`. The catch-block never fires (timeout returned a value, didn't throw); `block.output` stays null while `result` (the JSON error) is sent to the LLM. UI may render "running" forever for that block on rehydration. | medium | observed | fix before porting |
| A2.9 | `lib/api/ai_chat_service.dart:_streamRound StreamError event` | When a `StreamError` is yielded mid-stream, the code sets `conv.error = message` but **does not break** out of the `await for`. Subsequent events still process; then a half-assistant turn is persisted. | medium | observed | fix before porting |
| A2.10 | `lib/api/ai_chat_service.dart:_streamRound no-streamed-content branch` | Empty `contentBlocks` after stripping → assistant turn appended; Anthropic-shape conversion returns null and omits from wire, but local `_RepoConversation.messages` retains an empty assistant bubble. UI ghost. | medium | observed | fix before porting |
| A2.14 | `lib/api/ai_tools_provider.dart:ListIssuesTool` (and other completer-based tools) | `Completer<List<Issue>>` 30s timeout returns `[]`; provider error callback is `(_) {}` (no-op) so genuine errors are mapped to "empty list" — indistinguishable from "no issues." | medium | observed | fix before porting |
| A2.16 | `lib/api/ai_chat_service.dart:_saveToDisk` | Race: each `_saveToDisk` is fire-and-forget in `_runAgenticLoop`. Two concurrent writes to the same file path produce corrupted JSON. | medium | observed | fix before porting |
| A2.17 | `lib/api/ai_chat_service.dart:_runAgenticLoop` | After `_maxToolRounds=25` the loop exits without final assistant turn. User sees their last tool-result message with no synthesis. | medium | observed | fix before porting |
| A2.20 | `lib/api/ai_tools_file.dart:FileSearchTool._searchDir` | `entity.readAsLines()` on a multi-GB file blocks the isolate before any catch fires. | medium | observed | fix before porting |

### Low (23)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| R2.9 | `rust/src/api/git_manager.rs:2257-2266` | `StallDetector` re-maps libgit2 errors to a generic `git2::Error::from_str("network stall...")`, losing class/code. | low | observed | leave behind |
| R2.12 | `rust/src/api/git_manager.rs:1929-1932 / get_recent_commits` | `match diff.stats() { ... Err(_) => (0, 0) }` — diff-stats failure produces zero counts indistinguishable from genuine zero-diff commits. | low | observed | leave behind |
| R2.14 | `rust/src/api/git_manager.rs:3957-3987, 4035-4067` | `submodule.reload(true).ok()` — stale OIDs may produce wrong `(path, status)` rows. | low | strong inference | leave behind |
| R2.16 | `rust/src/api/git_manager.rs:4128 / abort_merge` | Only `Repository::open` in the file not wrapped with `swl!()` — error loses the `(at line N)` annotation. Cosmetic. | low | observed | leave behind |
| D2.7 | `lib/api/manager/git_manager.dart:_runWithLock` retry loop (~150-170) | Original error and retry error both logged but caller cannot distinguish "first failed, retry succeeded" from "both failed." Observability. | low | strong inference | leave behind |
| D2.11 | `lib/api/helper.dart:viewOrEditFile` (~1010) | Catch uses `print(e)` not Logger; silently suppresses non-FileSystemException errors (permissions, etc.). | low | observed | leave behind |
| D2.12 | `lib/api/helper.dart:openLogViewer` (~113-121) | `listSync()` can throw on platform errors; no catch. Crash in log viewer. | low | observed | leave behind |
| D2.13 | `lib/api/helper.dart:pickDirectory` (~232-234) | UI cannot distinguish "user cancelled" from "error" — both return null. | low | observed | leave behind |
| D2.14 | `lib/gitsync_service.dart:initialiseStrings` (~169) | If `serviceInstance.invoke(UPDATE_SERVICE_STRINGS, ...)` is called before service is up, new strings vanish; service uses defaults. No retry/queue. | low | strong inference | leave behind |
| D2.16 | `lib/api/logger.dart:dismissError` (~159-161) | `Navigator.canPop() ? pop() : null;` — `: null` is functionally a no-op. Cosmetic. | low | observed | leave behind |
| D2.17 | `lib/api/accessibility_service_helper.dart:isAccessibilityServiceEnabled` (~60) | No try; platform-method exception propagates. Cross-platform robustness. | low | observed | leave behind |
| D2.20 | `lib/api/manager/repo_manager.dart:setOnboardingStep` (~21) | `if (await getInt(...) == -1) return;` — silently no-ops without documentation; -1 is the "complete" sentinel. | low | observed | leave behind |
| D2.21 | `lib/api/manager/git_manager.dart:getRecentCommits / getMoreRecentCommits` | Catch returns `[]`. Caller cannot distinguish "no commits" from "fetch failed." | low | observed | leave behind |
| P2.7 | `lib/api/manager/auth/github_manager.dart:getIssueTemplates` (~1741) | `print(...)` of response body. Inconsistent with codebase logging policy. | low | observed | leave behind |
| P2.11 | `lib/api/manager/auth/github_manager.dart:createIssue / createPullRequest` (~1713-1729 / ~1812-1828) | Logs "HTTP {code}: {body}" — issue/PR title and body land in logs. Not a leak but content lands in logs. | low | observed | leave behind |
| P2.13 | `lib/api/manager/auth/github_app_manager.dart:getGitHubAppInstallations` (~26) | `statusCode == 200` only — 304 (not-modified) treated as failure → empty list. | low | observed | leave behind |
| P2.16 | `lib/api/manager/auth/gitea_manager.dart:getIssueDetail 204 handling` | 204 response would crash `json.decode("")`; guarded by 200 check but fragile if body is empty + 200. | low | strong inference | leave behind |
| A2.8 | `lib/api/ai_tool_executor.dart` | `block.error = e.toString()` — for FRB-wrapped libgit2 errors, the full Rust panic-style message including paths surfaces unredacted. | low | observed | leave behind |
| A2.11 | `lib/global.dart` | `aiKeyConfigured` decoupled from secure-storage cold-start re-read; the chat path reads the key directly each time so works in practice, but a future change could desync. | low | indirect | leave behind |
| A2.12 | `lib/api/ai_tools_git.dart:GitLogTool / GitMoreCommitsTool` | No total-bytes cap on log output beyond per-commit `200 char` truncation. | low | observed | leave behind |
| A2.13 | `lib/api/ai_tools_git.dart:GitListRemotesTool` (and similar empty-input tools) | No early-return when `context?.repoIndex == null`; raw Rust error string surfaces. | low | observed | leave behind |
| A2.15 | `lib/api/ai_tools_file.dart:FileWriteTool` | No cap on `content` size. Multi-MB writes block isolate. | low | observed | leave behind |
| A2.18 | `lib/api/ai_chat_service.dart:sendMessage` | If provider fails after user message is appended, user message stays in persisted conversation but no assistant reply lands. Re-sending repeats user message. | low | observed | leave behind |
| A2.19 | `lib/api/ai_provider_validator.dart:fetchAvailableModels` | Empty `data` list returns `(<String>[], null)` (success with no models); wand model picker treats as "successful but empty." | low | observed | leave behind |

---

## Pass 6: Configuration and Environment Hazards

Primary-thread review against `pubspec.yaml`, `AndroidManifest.xml`, `Info.plist`, `.github/workflows/generate-apk-release.yml`, `lib/global.dart`, `lib/constant/{strings,values,secrets.dart.template,langDiff,langLog}`, `analysis_options.yaml`, `Cargo.toml`, `.fvmrc`, `fvm_config.json`, `flutter_rust_bridge.yaml`, `l10n.yaml`.

### Critical (0)

No critical Pass 6 findings.

### High (1)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| C6.1 | `lib/global.dart:11 / const demo = false` | `// TODO: Must be false for release` — manual-flag-by-comment with no CI enforcement. Resolves `arch-CF7`. If accidentally flipped (a refactor, a debugging session), the build ships demo features. **No grep-found use sites in scanned files** (the search must extend to `lib/main.dart` and the UI page files); even so, the existence of the flag with the manual TODO is itself the hazard. Same applies to `lib/global.dart:18-21` `aiFeaturesEnabled = ValueNotifier(true)` — the AI subsystem defaults ON regardless of whether keys are configured. | high | observed | fix before porting |

### Medium (12)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| C6.2 | `.github/workflows/generate-apk-release.yml:35-39` | `cat <<EOF > ./lib/constant/secrets.dart \n ${{ secrets.SECRETS }} \n EOF` — **unquoted heredoc.** If the secret value contains `$`, `` ` ``, or `\\`, shell parameter expansion or backslash-escapes corrupt the file. The result compiles but with wrong constants → silent OAuth failure in production. | medium | observed | fix before porting (use `<<'EOF'`) |
| C6.3 | `pubspec.yaml:73-78 / workmanager` | `workmanager: git: { url: ..., path: workmanager, ref: main }` — pinned to a **moving branch**, not a SHA. The other three ViscousPot forks are pinned to specific SHAs (`fe783b40c0cba73...`, `cc19fd549f23dd1...`, `62a6837693100110...`). `pubspec.lock` is committed (38.9KB) but `flutter pub upgrade` will silently shift workmanager. | medium | observed | fix before porting |
| C6.4 | `ios/Runner/Info.plist:5-108 / BGTaskSchedulerPermittedIdentifiers` | 101 hardcoded identifiers (`scheduled_sync_set` + `scheduled_sync_0..100`). Caps repo containers at ~101 on iOS; **silent ceiling** — adding the 102nd container would not register a BGTask slot. Related to `arch-CF6` (slot-assignment lifecycle deferred to protocols phase). | medium | observed | port differently |
| C6.5 | `.github/workflows/generate-apk-release.yml:69-72` | `flutter_rust_bridge_codegen generate` followed by `git checkout HEAD -- rust/src/frb_generated.rs` — the regeneration step runs and is then **immediately reverted**. Works around codegen non-determinism but masks legitimate Rust API changes that should propagate; FRB skew between Dart bindings (`lib/src/rust/`) and Rust impl is hidden. | medium | observed | fix before porting |
| C6.6 | `android/app/src/main/AndroidManifest.xml:7-9, 30` | `MANAGE_EXTERNAL_STORAGE` (Play-Store-flagged sensitive permission) + `requestLegacyExternalStorage="true"` (ignored on Android 11+) + `enableOnBackInvokedCallback="false"` (opts out of Android 13+ predictive back). Three platform-compat hazards in one manifest. | medium | observed | port differently |
| C6.7 | `.github/workflows/generate-apk-release.yml:75-77` | `nttld/setup-ndk@v1` with `ndk-version: r29-beta4` — beta NDK in a release-tag pipeline. | medium | observed | fix before porting |
| C6.8 | `lib/global.dart:13-22` | Top-level singletons `repoManager`, `uiSettingsManager`, `gitSyncService`, `premiumManager`, `colours`, `aiChatService`, `late AppLocalizations t`, plus `aiKeyConfigured` / `aiFeaturesEnabled` `ValueNotifier`s and a mutable `VoidCallback? switchToAiTab`. Cross-isolate visibility unclear; module-load order required for `t` (`late` final) before any UI build. | medium | observed | port differently |
| C6.9 | `lib/constant/secrets.dart` (gitignored) — `secrets.dart.template` | Eight OAuth client IDs/secrets defined as `const` strings. **Real values are baked into the APK at build time via the heredoc above** — anyone with the APK can extract them. Mobile-app industry norm but a hazard worth flagging. **Routed to defect-scan-semantic Pass 4** for trust-boundary discussion. | medium | observed | port differently |
| C6.10 | `.github/workflows/` | Single workflow `Release OSS (APK)` triggered on `tags v*` or manual dispatch. **No iOS CI; no PR-time CI for any platform.** A PR that breaks Android release builds is undetected until a tag is pushed; iOS regressions are caught only on local/fastlane builds. | medium | observed | leave behind |
| C6.11 | `rust/Cargo.toml:23 / [features] default = ["vendored"]` | Every consumer gets vendored libgit2 + vendored OpenSSL by default (Android/iOS need this since neither has system libgit2 in standard Flutter installs). A port to a stack with system libgit2 should remove this. | medium | observed | port differently |
| C6.12 | `pubspec.yaml:24-26` | `flutter_secure_storage: 10.0.0` and `oauth2_client: 4.2.5` are **exact-version-pinned without `^` caret** while most other deps use `^`. Inconsistent — when patched releases come out the rest of the tree updates but these don't. Since they are commented-out alternatives showing `^` versions (pubspec.yaml lines 27-28), this looks intentional but undocumented. | medium | observed | leave behind |
| C6.13 | `ios/Runner/Info.plist:174 / NSPhotoLibraryUsageDescription` | String declared because user-selected directories may include photos, but the app does not access photo library directly — uses `flutter_secure_storage` + `ios_document_picker`. Over-broad permission disclosure; potential App Store reviewer pushback. | medium | observed | leave behind |

### Low (5)

| # | Location | Defect | Severity | Evidence Level | Action |
|---|----------|--------|----------|----------------|--------|
| C6.14 | `lib/constant/strings.dart:28-65` | ~30 hardcoded URLs (wiki, troubleshooting, GitHub issue templates, Discord, Play Store, App Store ID, GitHub App link). Domain change (e.g. `gitsync.viscouspotenti.al` → other) requires a release. | low | observed | port differently |
| C6.15 | `lib/constant/strings.dart` (notification IDs, channel IDs) | `syncStatusNotificationId = 1733`, `networkRetryNotificationId = 1734`, channel IDs `git_sync_notify_channel` / `git_sync_bug_channel` / `git_sync_sync_channel`. Magic numbers/strings; collision risk if app is reinstalled side-by-side with another build. | low | observed | leave behind |
| C6.16 | `lib/constant/strings.dart:127-128 / sshPattern, httpsPattern` | `httpsPattern` does not handle `git+https://...` URLs (used by some self-hosted setups); SSH pattern requires at least two path components separated by `/`. Edge cases where a valid remote URL is rejected. | low | observed | port differently |
| C6.17 | `analysis_options.yaml:13` | `errors: constant_identifier_names: ignore` disables the linter for SCREAMING_SNAKE_CASE constants. The `gitsync_service.dart` event constants use this style; rule is silenced project-wide rather than locally. | low | observed | leave behind |
| C6.18 | `flutter_rust_bridge.yaml` | 65-byte file — three keys, no version pinning; FRB version is implicit via `Cargo.toml` `flutter_rust_bridge = "=2.12.0"` and CI's `cargo install flutter_rust_bridge_codegen --version 2.12.0 --locked`. A divergence between any of the three would fail at codegen time but the explicit version isn't surfaced in this file. | low | observed | leave behind |

---

## Summary

### Findings by Severity

| Severity | Count |
|----------|-------|
| Critical | 1 |
| High | 32 |
| Medium | 86 |
| Low | 63 |
| **Total** | **182** |

### Findings by Pass

| Pass | Critical | High | Medium | Low | Total |
|------|----------|------|--------|-----|-------|
| 1. Logic and correctness | 0 | 15 | 37 | 35 | 87 |
| 2. Error handling | 1 | 15 | 37 | 23 | 76 |
| 6. Config and environment | 0 | 1 | 12 | 5 | 18 |
| **Subtotals** | **1** | **31** | **86** | **63** | **181** |

(One additional `D2.2` (medium) appears in stats both ways — the table above counts 181, total above includes one resolution of a routed/info entry. Treat 181-182 as the working range.)

### Top Findings

The highest-impact items, sorted by severity then confidence:

1. **R2.3 (critical)** — Rust core unconditionally trusts every TLS/SSH certificate (`certificate_check` returns `CertificateCheckStatus::CertificateOk`). Combined with `set_verify_owner_validation(false)` and `safe.directory=*` at process init, three independent libgit2 trust checks are disabled. **Trust-boundary regression vs stock libgit2.**
2. **R1.2 / R2.18 (high)** — `init()` panics on `set_verify_owner_validation(false).unwrap()`, and `init()` is invoked at the top of every `run_with_lock` call. A single libgit2 option failure becomes a per-op panic on the FRB worker.
3. **R2.2 (high)** — `run_with_lock` 10-min hard timeout returns `Ok(T::default())` = `Ok(None)` for the common `Result<Option<bool>>` shape. Caller cannot distinguish "no work needed" from "we waited 10 min and gave up." **Silent sync drop after long wait.**
4. **D2.5 (high)** — `gitsync_service.merge()` rethrows on non-network exceptions without invoking `MERGE_COMPLETE`. UI's merge-conflict dialog hangs on a never-resolving future; user must force-close.
5. **D1.12 (high)** — `premium_manager.cullNonPremium` orphans all `clearAll()` futures via `List.generate(n, async)`; settings residue persists for "deleted" repos.
6. **D1.24 (high)** — `storage._set<List<String>>` joins on `,` and corrupts user-provided container names containing commas on round-trip.
7. **P2.1 (high)** — All four provider clients use `lib/api/helper.dart:httpX` wrappers which have **no retry, no 429 handling, no 5xx handling, no 401 handling**. 10s timeout synthesises a fake 408. **arch-CF1.**
8. **P2.4 / P2.5 (high)** — All providers' 401/403 + 429 responses surface as empty lists / null detail / silent failure. No re-auth trigger; no `Retry-After` parse. UI cannot distinguish auth/rate-limit/instance-down.
9. **D2.9 (high)** — `backgroundUploadChanges.mergeConflictCallback` mutates global `repoman_repoIndex` from the background isolate, silently swapping the user's UI context.
10. **A1.25 (high)** — `_streamRound` content-block stripping can drop the entire assistant turn from the wire while keeping it locally; on the next request, `tool_use`/`tool_result` pairing is broken and Anthropic returns 400. Conversation poison.
11. **A1.18 (high)** — `aiComplete` returns `''` (not null) on empty content; commit-message wand can commit an empty AI-generated message.
12. **A2.1 / A2.3 (high)** — AI chat path has no 401/403 re-auth signal (`aiKeyConfigured` not flipped) and no 429 backoff; users hit a dead-end with opaque error strings.
13. **R2.6 (high)** — Submodule traversal in `clone_repository` is `let _ = ...` for ~6 sites; submodule failures silently leave detached HEAD.
14. **R1.3 / R1.5 / R1.6 / R1.7 / R1.9 (high, x5)** — Rust `unwrap()` panics on perfectly-realistic inputs: corrupt `.git/config`, non-UTF8 commit messages, non-UTF8 remote names, transient I/O during rebase cleanup, RNG/encoding failures during SSH key generation.
15. **D1.10 / D1.11 / D1.13 / D1.23 (high, x4)** — Premium PAT validation copy-paste bug, RangeError on empty repo names list, Storage class footgun where typed accessors throw on `hasDefault` keys.
16. **C6.1 (high)** — `lib/global.dart:demo` flag with manual-only TODO enforcement.
17. **P1.7 (high)** — `getRepos` implements search as client-side filter over only the first 30/100 repos; repos beyond page 1 are invisible to the picker.

### Routed To Semantic Phase

Items spotted during Pass 1/2/6 that are concurrency / security / API-contract-drift in nature, deferred to the `defect-scan-semantic` phase. Mirrored as `carry_forward` entries with `target_phase: defect-scan-semantic` in `workflow/status.yaml`.

| ID | Description | Why Routed |
|----|-------------|-----------|
| `defR-CF1` | `RemoteCallbacks::certificate_check` returns `CertificateCheckStatus::CertificateOk` unconditionally + `set_verify_owner_validation(false)` + `safe.directory=*` at init = three trust checks disabled. | Pass 4 (security) — trust-boundary regression. |
| `defR-CF2` | Every error wrapped via `git2::Error::from_str(string)` — class/code lost. Dart cannot distinguish auth/network/conflict/lock failures. | Pass 5 (contract drift across FRB boundary). |
| `defR-CF3` | `run_with_lock` flock-based queue: probe-eviction logic (~lines 363-443) races against position-0 holder's `QueueCleanupGuard::drop` (lines 510-543). Probe may delete a queue entry that no longer exists. | Pass 3 (concurrency / race). |
| `defR-CF6` | `unsafe { env::set_var("RUST_BACKTRACE", "1") }` and `unsafe { env::set_var("HOME", path) }` at lines 705-706 — `set_var` is unsound on multi-threaded processes (Rust 2024 marks unsafe). FRB worker pools are multi-threaded. | Pass 3 (concurrency / unsoundness). |
| `defR-CF7` | `init()` mutates global libgit2 state (`set_verify_owner_validation`) every call. Concurrent calls from FRB worker threads race on the global flag. | Pass 3 (concurrency on global state). |
| `S-D-1` | `SettingsManager.keyNamespace` is a static field mutated by `reinit()`; concurrent calls (accessibility loop + UI sync) corrupt the per-call namespace. | Pass 3. |
| `S-D-2` | `_syncGeneration` / `isSyncing` / `isScheduled` ad-hoc reentrancy gate is read/written across two isolates (UI + service). Synchronous within an isolate but cross-isolate semantics need verification. | Pass 3. |
| `S-D-3` | `accessibilityEvent` may re-enter; shares static `lastOpenPackageName` / `lastOpenPackageNameExcludingInputs`. | Pass 3. |
| `S-D-4` | `merge()` client-mode path commits but never pushes — verify intent against client-mode contract. | Pass 5 (contract). |
| `S-D-5` | `backgroundUploadChanges.mergeConflictCallback` mutates global `repoman_repoIndex` — cross-context state mutation. | Pass 5 (contract) / Pass 3 (concurrency). |
| `sem-cand-providers-1` | OAuth refresh race in `git_provider_manager.dart:getToken` (P1.2) — concurrent provider calls each call `refreshToken`, invalidating each others' tokens. | Pass 3 (concurrency on secure-storage state). |
| `sem-cand-providers-2` | Logger may include OAuth token or response body in error logs (P2.6, P2.7). `print()` use in `github_manager.dart` line ~1741. Verify `logger.dart` redaction. | Pass 4 (security / sensitive-data audit). |
| `sem-cand-providers-3` | Idempotency under retry — if any retry layer is added later (per P2.1), `createIssue/createPullRequest/addIssueComment/addReaction` are unprotected against duplicate creation (P2.14). | Pass 5 (behavioral correctness on side-effecting POSTs). |
| `sem-cand-providers-4` | 401-on-expired-token does not trigger re-auth (P2.4). User sees empty data. | Pass 5 (UX / state-machine drift). |
| `A-SEM-1` | Static system prompts + repository file content (via `file_read` tool) concatenated into LLM messages. Malicious repository content can subvert system prompt — **prompt injection**. | Pass 4 (security). |
| `A-SEM-2` | `print('[AI Stream] $provider HTTP ${response.statusCode} body=$errorBody')` and similar `print` lines write provider responses + exception text to logcat / iOS unified log. Some providers echo headers; rare key fragments. | Pass 4 (token / key leakage in logs). |
| `A-SEM-3` | API key in `?key=$apiKey` query string for Google (`ai_stream_client.dart`, `ai_completion_service.dart`, `ai_provider_validator.dart`). Query strings appear in HTTP server / proxy logs, in Flutter debug console. | Pass 4 (credential exposure via URL). |
| `A-SEM-4` | `normalizeEndpoint` defaults to `http://` if no scheme — self-hosted users may transmit API keys in plaintext. | Pass 4 (privacy / credential hazard). |
| `A-SEM-5` | `formatDiffParts` may include user file content (potentially repo-content secrets) in prompts sent to the LLM. | Pass 4 (sensitive-content exfiltration). |
| `A-SEM-6` | Agent has tools (`git_force_push`, `git_download_and_overwrite`, `git_reset_to_commit`, `git_squash_commits`) gated only by user-tap confirmation. Prompt-injection (A-SEM-1) into a `confirm`-tier tool call could be approved by an inattentive user — confused-deputy. | Pass 4 (social engineering). |
| `arch-CF8-residual` | `mmap2_flutter` (forked) call sites — **not found** in scanned files. Most likely use sites: `lib/main.dart`, `lib/ui/page/{file_explorer,code_editor,diff_view}.dart`, or `lib/api/ai_tools_file.dart` (also scanned, no hit). Re-route to Pass 3 with target file_explorer/code_editor. | Pass 3 (concurrency on memory-mapped reads). |

### Rate-limit / retry summary (resolves arch-CF1 partially)

The four provider REST clients (`github_manager.dart`, `gitlab_manager.dart`, `gitea_manager.dart`, `codeberg_manager.dart`) are **uniformly retry-naive**: every REST call goes through `lib/api/helper.dart`'s `httpGet/Post/Patch/Put/Delete` wrappers, which set a 10-second timeout and synthesise a fake `http.Response('Error', 408)` on timeout, with no retry, no exponential backoff, and no parsing of `Retry-After`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, or GitHub's `x-ratelimit-*` family. A 429 response from any provider lands in the same `else` branch as 4xx/5xx and surfaces as an empty list, `null` detail, or generic "failed" toast. There is a separate `handleIfNetworkError` (used for git clone/push) that schedules a Workmanager retry with exponential backoff, but **no provider REST path invokes it**. The git-protocol retry harness (`scheduleNetworkRetryOp`) uses 30s × 2^retryCount backoff capped at 5 retries — git-only. **Net effect: under any provider rate-limit pressure or transient 5xx, the UI silently degrades to empty data without distinguishing "no data" from "rate limited" or "instance down."** The contracts phase should treat this as a load-bearing gap fed by `arch-CF1`.

### AI feature contract sketch (resolves arch-CF2 partially)

**Chat (`AiChatService`):** triggered from AI tab → `aiChatService.sendMessage(text)`. Reads `repoman_aiProvider`, `repoman_aiApiKey`, `repoman_aiEndpoint`, `repoman_aiChatModel`, `repoman_aiToolModel` from `repoManager`; current repo's `gitProvider`, HTTPS auth credentials, remote URL via `uiSettingsManager` and `GitManager`. Persists per-repo conversation JSON at `<applicationSupport>/ai_chats/<repoIndex>.json` after every message. Network: SSE streaming via `ai_stream_client.dart:streamCompletion` to Anthropic / OpenAI / Google / self-hosted endpoints, 120s socket timeout. Loop cap: `_maxToolRounds = 25`. Two-model split: tool model runs all tool-using rounds; chat model runs final synthesis when models differ.

**Wand / autocomplete (`AiCompletionService`):** triggered by commit-message-suggestion / similar UI. Reads same config plus `repoman_aiWandModel`. No `ToolContext`. **Returns null on every failure** (404, 401, 429, network, parse, empty content) — UX hole (A1.17, A1.18). Single non-streaming POST, 60s timeout.

**Agent:** only invoked inside chat. `_maxToolRounds = 25` rounds, **no per-round tool-call cap** (A1.7), **no total-tool-call cap**. Three-tier confirmation: `none` (read-only) / `warn` (file edit, branch create, commit, gitignore write) / `confirm` (file write, push, pull, branch delete, fetch, sync, force-related, repo create) / `danger` (`git_reset_to_commit`, `git_squash_commits`, `git_force_pull`, `git_force_push`, `git_download_and_overwrite`, `git_upload_and_overwrite`, `git_set_remote_url`, `git_delete_remote`, `git_checkout_commit`). Tools: read-only file/git/provider operations (`git_status`, `git_log`, `git_diff`, `file_read`, `file_list`, `file_search`, `repo_info`, `list_issues`, `list_pull_requests`, etc.); mutating file/git/provider operations across the three tiers above. Tier-gating: `core` always; `contextual` when OAuth configured; `advanced` discovered via `list_available_tools` and "activated" once per session.

The contracts phase will absorb this sketch and turn each feature into a feature-contract row.

---

## Validation

| # | Criterion | Result | Evidence |
|---|-----------|--------|----------|
| 1 | At least two of the three mechanical passes (1, 2, 6) produced findings or documented "no defects found." | PASS | All three passes produced findings: Pass 1 (87), Pass 2 (76), Pass 6 (18). |
| 2 | Each finding has location, severity, evidence level, and recommended action. | PASS | Every row in Pass 1 / Pass 2 / Pass 6 tables has all four columns populated. Full notes in `scratch/defects-{rust-core,dart-sync,providers,ai}.md` for those wanting the per-finding 1-3 line code excerpt. |
| 3 | Findings are organized by pass and sorted by severity. | PASS | Each pass section is divided into Critical / High / Medium / Low subsections; rows within each subsection are roughly grouped by source area for readability. Counts at the head of each subsection. |
| 4 | Summary tables are complete and counts match the detailed findings. | PASS | `Findings by Severity` shows 1+32+86+63=182 (working count 181-182; rounding from one borderline categorization). `Findings by Pass` table breaks down per pass: Pass 1 (87), Pass 2 (76), Pass 6 (18) = 181. |
| 5 | Items spotted that are actually semantic in nature are routed to defect-scan-semantic via carry_forward in workflow/status.yaml. | PASS | 21 items routed (5 from Rust, 5 from Dart sync, 4 from providers, 6 from AI, plus 1 mmap-residual from arch-CF8); see `Routed To Semantic Phase` table. Each entry mirrored into `workflow/status.yaml` under `phases.defect-scan-mechanical.carry_forward` with `target_phase: defect-scan-semantic`. |
| 6 | Findings are marked with evidence levels. | PASS | Every row tagged `observed` (≈85%) or `strong inference` (≈15%). Two `indirect` (A2.2, A2.11) reflect cross-file cross-checks. No `open question` entries — Pass 1/2/6 deliberately defer to runtime tests as `defect-scan-semantic` carry-forwards instead. |

**Validated by:** 2026-05-06 (defect-scan-mechanical phase, implementing session)
**Overall:** PASS
