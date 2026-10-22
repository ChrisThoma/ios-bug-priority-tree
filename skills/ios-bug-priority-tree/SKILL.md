---
name: ios-bug-priority-tree
description: Journey-centered finder tracks, severity, and escalation for an automated iOS bug hunt. Load at the START of the hunt to discover and test user journeys for UI functionality, understandable state, consistency, recovery, edge cases, and measurable user impact while preserving Track A-D orchestration.
---

# iOS bug hunt: user journeys, impact, and escalation

Companion to a "find N distinct issues" iOS goal protocol. Load this skill before the first finder wave. The host protocol owns turns, builds, commits, issue status, fixes, and verifier scheduling. This skill owns app-map guidance, finder methods, severity, evidence, wave composition, and per-track escalation.

The four track names are a stable interface. Keep Track A through Track D in prompts, ledgers, resume logic, and `HUNT.md` forwarding. The accessibility tree is a simulator navigation and observation tool; using it does not make the hunt an accessibility audit. Accessibility-specific defect hunting is a late escalation unless the host prompt explicitly requests it earlier.

## Host setup

The skill is host-agnostic. Install it at `~/.claude/skills/ios-bug-priority-tree/` for Claude Code or `~/.codex/skills/ios-bug-priority-tree/` for Codex. Per-project skill directories work too.

Baseline requirements:

- A simulator-driving server exposing targeted tap, text entry, element lookup, tree description, and screenshot calls. The examples use `ui_find_element` and `ui_describe_all`; substitute equivalents.
- `xcrun simctl` and `sips` on PATH.
- Fresh subagents for finder waves and verification.

Conditional tools:

- `ffmpeg` for motion analysis and video-based timing.
- `idb` when the selected timing procedure uses it.
- `leaks` and `footprint` for targeted memory investigation.

Every simulator-bound prompt names exactly one UDID. Every simulator and `simctl` call must target that UDID explicitly; never use an implicit `booted` target when multiple simulators exist. Give concurrent workers unique artifact directories and filenames.

## Safety and test contract

Before hunting, establish what can be inferred from the repository and host prompt:

- Supported target, deployment target, installed simulator runtimes, orientations, app capabilities, account roles, and integrations.
- Test account, backend, fixtures, reset method, and whether destructive actions are safe.
- Product promises from requirements, tests, visible UI copy, and established behavior elsewhere in the app.
- Known limitations, unsupported conditions, and environment constraints.

Use sandbox accounts and synthetic data. Do not make real purchases, mutate production data, expose credentials or personal data in evidence, or perform irreversible external actions without explicit authorization. Host-wide network changes and other disruptive manual conditions require user approval and a restoration plan.

If the simulators recorded in the map no longer exist, recreate them under the same names and device types, record the new UDIDs and runtime, and say so in the first dispatch delta. If no installed runtime is within one major version of the deployment target, mark minimum-OS coverage `unsupported` in the map and tell the user once before wave 1: fixes that rely on newer-OS APIs cannot be verified for the oldest supported users on this machine.

A finding's expected behavior must be grounded in a visible UI promise, documented requirement, data invariant, established equivalent behavior, or strong iOS convention. If the expected behavior is genuinely ambiguous, report a `PRODUCT QUESTION` lead rather than a candidate. Unreachable functionality is a coverage gap, not a defect, unless its entry point should be available to the tested user.

## APPMAP.md: journey map

Create `APPMAP.md` before wave 1 from bounded source inspection, app metadata, launch behavior, and supplied product information; runtime finders then return concise deltas when they discover new paths or prerequisites. If a map already exists, validate and migrate it to the journey schema below before reuse. Do not treat a screen list as a journey map. The orchestrator is the single writer.

Keep `APPMAP.md` compact and structural. It describes the app, not the hunt: it holds no coverage state, issue numbers, verdicts, wave status, installed-build revisions, fix commit hashes, or hunt-relative wording such as "this hunt" or "this session." Coverage lives in the ledger's `## Coverage` section and installed builds in its `## Builds` section (both below); defect status lives in the ledger's issue rows. Blind verifiers read the same map as every other worker, so anything a verifier must not see does not belong in it. A map may cite a prior hunt's finding only as `round-N #M` for an intentional-behavior or dead-lead note. One Target per map; never widen the target mid-hunt.

When migrating or trimming a map, these survive every migration and are restated, never deleted: launch/reset commands, fixtures and test-data sentinels, capabilities and blocked entry points, intentional-behavior notes, tooling caveats and traps, `unsupported` markers, and dead leads that were investigated and disproven. An installed-revision line found in an existing map moves to the ledger's `## Builds` section.

Give every subagent the map atomically. In a shared workspace, prefer an explicit first action to read the exact repo-root `APPMAP.md`; otherwise paste its full contents. Never use placeholders, summaries, ellipses, or "same as above." Before dispatch, reject any prompt containing an unexpanded map/track placeholder. A worker must read the map before simulator or source work and identify its assigned JID.

For each meaningful journey, record:

```
JID | user goal | entry path | prerequisites/test data | expected outcome
states/branches: success, validation, cancel, loading, empty, error, retry, completed (applicable only)
notes: intentional behavior, traps, and launch arguments specific to this journey
```

Also record exact launch/reset commands, the per-simulator environment baseline (locale, appearance, content size) with the commands that read each value back from inside the simulator (a host-side `defaults` read reports the Mac, not the simulator), the data-container reset to a named fixture state, the fixture import command and file shape, failure-injection recipes that probes reuse (an unwritable store, a malformed file), stable screen names, global navigation, test-data sentinels, capabilities, and blocked entry points. A journey is a user goal or meaningful task, not a screen or test condition. Visiting a screen does not cover its actions or states. Dynamic Type, contrast, appearance, locale/RTL, orientation, device size, and similar conditions are never standalone JIDs.

Finder reports include only map deltas (new or corrected journeys, prerequisites, traps, blockers) and coverage deltas (below). Do not return a rewritten map. Broad discovery is exhausted only when the known frontier is empty and fresh runtime passes plus bounded source inspection reveal no new meaningful journey or high-risk branch.

## Coverage: hunt state in the ledger

Coverage is hunt state, so it lives in the host ledger (`ISSUES.md`), under a `## Coverage` heading, one pipe-delimited line per mapped journey:

```
J1 | coverage: discovered | tested: (none) | next: primary path through the naming offer
```

States: `discovered | attempted | core-covered | blocked | unsupported`. Conditions exercised (Dynamic Type, locale/RTL, appearance, orientation, device size) go in `tested:` under the journeys they touched; they never substitute for journey coverage. A journey is `core-covered` only after its principal outcome and applicable state, cancellation, error, and persistence behavior have been observed.

The section is created at turn 1 with every mapped journey `discovered`, carried unchanged on resume, and never copied into `APPMAP.md`. Finder reports return coverage deltas as `J<n>: <state> | tested: +<condition> | next: <probe>`; the orchestrator applies them after each wave. To keep the host dashboard parser safe: coverage rows never carry a `status:` label, never sit in a markdown table, and never contain a literal `|` inside prose.

## Builds: installed revisions in the ledger

Which build each simulator carries is hunt state too. After `## Coverage`, the ledger keeps a `## Builds` section with one line per simulator slot:

```
SIM-A | build: build/Fix3/Chiptune.app | revision: 89b97cd
```

Update the line whenever a build is installed on that slot. Every simulator dispatch states the app path installed on its UDID from this section; finder and fixer prompts may add the revision, verifier prompts carry the path only. Never describe a build as current unless its revision matches HEAD. When a fix is verified, install that build on the other simulator before its next finder, so finders do not hunt on a build that still has fixed defects. The same parser rules apply as for coverage rows: no `status:` label, no markdown table, no literal `|` in prose.

## Repository priors and precedence

Use `HUNT.md` or equivalent repository priors to specialize the skill, not to silently replace its purpose. Product facts, known intentional behavior, safety constraints, capability limits, build traps, proven dead ends, and defect-specific verifier conditions take precedence over generic skill assumptions.

Priors documents must carry a fixer-facing section: build and test traps, CI gates, retry-chain tar pits with their commit hashes, and repo tooling a fixer needs are forwarded to fixers, never held as orchestrator-only. Tooling traps that produced a rejected candidate (tap latency, an empty accessibility tree, a stale simulator locale) belong in the section every finder receives. Cross-references to a prior hunt's issues use `round-N #M`; bare `#N` refers only to the current ledger.

Historical bug yield is assignment evidence, not a standing waiver of journey-first ordering. Track weights, wave-one mandates, or instructions that move accessibility-specific checks ahead of core UX/functionality apply only when the user or current host prompt explicitly requests that objective for this target. Imperative wording or yield statistics in a mined `HUNT.md` do not make it current. Otherwise retain them as leads for the applicable escalation tier. When a prior conflicts with the current host objective, the current objective wins; record the conflict rather than combining contradictory instructions.

A priors document that contains wave mandates, lacks a fixer-facing section, or cites prior issues as bare `#N` was mined under older rules. Follow this skill's ordering, forward its build traps and tar pits to fixers anyway, read its bare `#N` as a prior round, and tell the user once, in the first dispatch delta, that the priors need re-mining. Do not restamp such a document's header to the current HEAD; a fresh stamp on old content hides the staleness.

## Agent budgets - include this block verbatim in every subagent prompt

> **Budget.** Context cost grows with every turn you take. Work within these caps and return early rather than push past them:
> - **Finders:** return after **about 30 simulator interactions or 3 candidate issues**, whichever comes first. Each simulator finder owns one primary JID and at most one adjacent branch; complete that journey rather than skimming several. If setup consumes most of the budget, return the exact checkpoint and remaining probes. If you cannot reach an assigned path within 10 actions beyond its documented setup, record it as blocked and move on.
> - **Hard stop:** at **60 tool calls total** (finders) or **40** (verifiers and confirm agents), stop and return the report with whatever you have, even mid-probe. Keep a running tally of every call as you work, not an estimate at the end. No finding justifies passing this line; a partial report is the expected outcome, not a failure.
> - **Setup cost:** if reaching a required state (a large arrangement, many records, a full library) would take more than **20 actions** beyond the documented reset, do not tap it into existence. Return `FIXTURE NEEDED: <state>` with the exact file or data shape, so the host can inject it into the app container before the next pass.
> - **Verifiers and confirm agents:** the defect repro itself must complete in **15 simulator actions or fewer**, excluding documented fixture setup. If it does not, return `BLOCKED: could not complete repro path` with what you tried. Do not keep exploring to make the repro work.
> - **Fixers:** return after **80 tool calls or 25 minutes** with the diff, the test result, and what is still failing. Do not keep iterating past that line; a partial fix with an honest status beats a silent overrun. Repair the behavior within the surfaces named in the prompt. If the fix instead clamps, scales down, hides, or disables something, or depends on an API newer than the deployment target, say so in the report and name who loses what. Smoke-test on the device where the defect appears, or state that you did not. Before changing code, check the evidence and `APPMAP.md`: if the evidence shows no defect, or the map marks the behavior as intended, commit nothing and return `NO DEFECT: <reason>` so the host can reject the issue. A fix for a misread repro changes intended behavior and has to be reverted later. Tests you add fail instead of crashing: unwrap with `try XCTUnwrap` or `#require` before indexing, so a wrong count during the change is a failed assertion, not a crash report the user mistakes for an app crash. StoreKit tests use `SKTestSession` with the project's `.storekit` file, never the real store, and every `xcodebuild test` passes `-collect-test-diagnostics never` unless the host asked for diagnostics.
> - **Navigation:** use the assigned UDID and the simulator server's find-element call (`ui_find_element`, or describe a point) to locate controls. A full tree dump (`ui_describe_all`) is allowed at most **once per new screen state**, never after every tap.
> - **Act and look in one call:** when an action needs a visual check, put the action, a short wait, the screenshot, and the downscale in a single shell command (for example `idb ui tap --udid <UDID> <x> <y>; sleep 1; xcrun simctl io <UDID> screenshot <orig>.png; sips -Z 750 <orig>.png --out <view>.png`), then view the downscaled copy once. A separate tool call for each of tap, screenshot, downscale, and view spends four calls per interaction and will not fit the hard stop.
> - **Screenshots:** retain the original evidence and view each image at most once. Take screenshots when the track's method requires them or to capture evidence, not as a general navigation aid.
> - **Evidence:** return one primary artifact per finding. A bounded before/after pair or short video/contact sheet is allowed when a single image cannot prove the claim.
> - **Report:** return a structured list with at most 3 candidates. For each: journey ID, preconditions, exact repro, expected basis, expected vs actual, user impact, reproduction count when practical, evidence, severity guess, and suspected duplicate. Add compact map and coverage deltas and worthwhile leads. Keep the report at **25 lines or fewer** and do not narrate process or dead ends. End it with `interactions: <n> | tool calls: <n>` from your tally. Return once: leave no background sleep, monitor, or shell running after the report.

The host sets the model explicitly on every dispatch (its default worker model for finders, verifiers, and first fixes; its escalation model for refixes). A dispatch that leaves the model unset has not applied the host's model policy.

Workers cannot be stopped mid-run on a call count, so the orchestrator checks afterwards: compare the tool-use count on each completion against the cap for that role. When a worker overran, name it in the next dispatch delta and narrow the next assignment for that journey (fewer branches, or a fixture built first) rather than repeating the same scope.

## Severity ladder

Assign severity from demonstrated user impact, not from the finder track or checklist category:

- **S1 - catastrophic:** crash on a consequential or normally reachable path, unrecoverable data loss, security/privacy exposure, or a severe unintended irreversible action.
- **S2 - task failure or harmful result:** a meaningful user goal is blocked; committed or displayed data is wrong; input is lost; an action produces duplicate or unintended effects; recovery requires abandoning the journey or repairing data outside its normal flow.
- **S3 - material UX failure:** the task remains possible, but the app misleads the user, obscures necessary state, behaves inconsistently enough to cause a wrong action, or cannot be operated reliably. Examples include indistinguishable loading/error/empty states, absent feedback that leaves a consequential action ambiguous, keyboard obstruction, meaningful truncation, recoverable lost progress, or an unreliable control.
- **S4 - polish:** visible inconsistency or aesthetic degradation that does not obscure meaning, change behavior, mislead the user, or reduce reliable operation.

Record reproduction confidence, journey importance, recoverability, and affected conditions separately from severity. A rubric match is not automatically S3: state the actual user consequence. Severity is fixed at confirmation from that stated consequence. Raising it later requires new evidence recorded in the issue row; how close the hunt is to its goal or turn limit is never a reason. The host goal may count S1-S3 plus at most two S4 findings. Deduplicate by demonstrated behavior and triggering condition first; merge by root cause only when evidence supports it.

## Candidate and evidence rules

Track A, B, and D candidates require runtime evidence from the installed build. Reproduce from a known baseline twice when practical; record results such as `2/2` or `1/3`. One clear reproduction may be sufficient for destructive S1 behavior that is unsafe to repeat. Tool failure, environment contamination, unsupported capabilities, and uncertain product intent are not candidates.

Harness artifacts are a named false-positive class. Before any "silent", "no feedback", "nothing happened", "stuck", or "wrong state after tap" observation becomes a candidate, repeat it once with the lowest-latency tap available (`idb ui tap --udid <UDID> <x> <y>`, or the server's equivalent) followed immediately by `xcrun simctl io <UDID> screenshot`, so a slow or misplaced automation tap cannot masquerade as an app defect. Never estimate when a tap landed: if timing matters, bracket the tap and any recording with wall-clock timestamps. Environment contamination (a stale locale, appearance, content size, or dirty container left by an earlier worker) is the other common source; when a candidate is rejected for either cause, record the root cause under the map's tooling caveats in the same turn so later finders inherit it.

A candidate reaches CONFIRMED only with an independent reproduction: a separate confirm agent, or the finder's own `2/2` with evidence for a stable, deterministic defect. A fixer is never dispatched from a single unreproduced report; the false positives that cost the most are the ones that reach a fixer and then a verifier.

Use evidence appropriate to the claim:

- Stable visual or state defect: screenshot.
- Lost or inconsistent state: bounded before/after pair with the same sentinel data.
- Transient interaction defect: short video or contact sheet with timestamps.
- Crash: full triggering sequence and crash-report reference; a return to Springboard is a crash lead until process exit or a crash report supports it.
- Performance: commands, conditions, all measurements, and summary statistic.
- Source lead: file and line or symbol plus a proposed user-visible repro; no screenshot required.

A Track C finding may be a candidate without driving a simulator only when it includes file/line evidence and a precise user-visible repro that a simulator finder can drive with existing launch arguments. Otherwise it is a lead. Security, privacy, silent data-integrity, or background-behavior risks that cannot meet the user-visible gate remain leads with category-appropriate supporting evidence.

## Finder tracks

Hand each finder the budget block, an atomic `APPMAP.md` read instruction or full map, host priors that apply to it, and only its own track section. Before dispatch, cost the assignment against the budget: a probe whose setup alone exceeds the finder cap is a fixture task, not a finder task. Build the fixture (a saved document, seeded store, or container file) with an off-simulator worker or a shell command, install it, and only then assign the probe. Never write a prompt that asks a finder to construct a large state by hand.

Prompts describe what to exercise and observe, not what the app must do. Any "must", "should", or "must not" in a finder prompt has to restate a promise from the map, the priors, or visible UI copy; otherwise phrase the probe as "report what X does." A candidate whose expected basis traces only to the orchestrator's own prompt wording is rejected with that cause named, and the cause is recorded for the priors' orchestrator section.

Every track reports user impact, map deltas, and coverage deltas using the common rules above.

### Track A - Journey and UI functionality

Drive meaningful user goals through the tree. Coverage is verified state transitions, not controls or screens visited.

For each assigned journey:

1. Establish the documented baseline and unique sentinel data.
2. Perform the primary action and observe immediate feedback.
3. Wait for a settled success or failure state and verify the promised outcome.
4. Check other visible representations of the result, such as list/detail, badge/count, search/filter, or summary.
5. Navigate away and return; relaunch when persistence or restoration is part of the product contract.
6. Exercise applicable validation, cancel/back, retry, destructive confirmation, repeated action, and interruption behavior.

Baked-in probes, applied only when relevant:

- Rapidly repeat consequential submit, save, purchase-sandbox, like, or destructive actions; judge duplication against the action's contract rather than assuming every repeat is invalid.
- Enter empty, invalid, minimum, maximum, duplicate, realistic long, and repeated data where the UI implies such boundaries.
- Begin an edit and separately test cancel and save. Check that each result appears consistently throughout the app.
- Once per stateful assigned area, background or terminate mid-flow and relaunch. Do not assume drafts must survive force termination unless the app promises restoration; look for corruption, false success, duplicate effects, or an unrecoverable state.
- If the app unexpectedly exits, inspect `~/Library/Logs/DiagnosticReports/` and replay the full sequence before classifying it as a crash.

### Track B - Visual-functional UX and consistency

Screenshots are the inspection method for this track. Navigate through actual journeys, then capture representative states rather than every screen. Inspect pixels together with observed interaction behavior.

Core rubric:

- **State clarity:** loading, empty, error, success, selected, disabled, destructive, and completed states are distinguishable and lead to a clear next action.
- **Action feedback:** consequential work produces perceivable, accurate feedback while in flight and at completion. A spinner or disabled button is not mandatory; optimistic updates, navigation, inline changes, and system feedback can be valid. File a candidate only when the absence or inaccuracy causes demonstrated ambiguity or harmful repetition.
- **Behavioral consistency:** equivalent actions use consistent labels, placement, gestures, confirmation, cancellation, and feedback. Compare equivalent concepts, not merely similar-looking controls.
- **Visual operability:** controls do not overlap, clip, shift out of reach, or become obscured by sheets, safe areas, or the keyboard. Visible target size alone does not prove the tappable frame; verify unreliable activation behavior before assigning S3.
- **Content integrity:** realistic long content remains understandable; truncation or wrapping does not change meaning or hide required actions.
- **Cross-state integrity:** transitions do not flash stale or wrong data, lose identity, jump to an unrelated position, or settle into an ambiguous state.
- **Appearance:** sample representative flagship states in light and dark mode. File issues when information or operation changes, not for preference alone.

```
xcrun simctl ui <UDID> appearance dark
xcrun simctl ui <UDID> appearance light
```

Capture keyboard-up evidence on input-heavy journeys and before/after evidence for state changes. Preserve full-resolution originals; inspect downscaled copies. Pure visual variation without user consequence is S4 or a product question.

### Track C - Source-assisted user-impact audit

Read source without driving a simulator. Use `APPMAP.md` to connect implementation risks to mapped journeys and to discover hidden capabilities, routes, prerequisites, and state branches.

Prioritize:

- Wrong data, boundary, date/timezone/locale, sorting/filtering, and persistence logic.
- Errors that are discarded, converted to false success, or leave loading state stuck.
- Races, duplicate requests, unawaited tasks, stale responses, and shared mutable state.
- One logical change split across multiple state systems, especially observable model state plus local view state.
- Force unwraps, `try!`, ignored save/commit results, and user-controlled indexing.
- Authentication/session, role, entitlement, import/export, sync, notification, deep-link, and declared capability paths applicable to the app.
- Silent security, privacy, analytics-consent, background-sync, and data-integrity failures.

A lead includes file/line or symbol, affected journey, trigger, predicted user consequence, and a concrete runtime probe. Ordinary leads become candidates only after user-visible runtime reproduction. Return hidden-route and prerequisite discoveries as `APPMAP.md` deltas. Pin reports to the source revision supplied by the host when fixes may occur concurrently.

### Track D - Targeted performance probe

Run after mapped journeys identify high-frequency interactions, visible stalls, expensive screens, or repeated flows worth measuring. Simulator results are triage evidence, not physical-device certification.

For interaction and launch timing:

- Define an observable start event and semantic completion condition before recording.
- Keep build type, simulator, fixture, cache/network state, and concurrent host load stable.
- Warm up once, collect at least three measured runs, and report all values plus median and worst case.
- Treat visible unresponsiveness over 1 second or cold launch over 2 seconds as investigation thresholds, not automatic defects.
- Scene-change detection may locate a window to inspect, but scene counts and gaps alone do not prove a hang.

```
xcrun simctl io <UDID> recordVideo --codec h264 --force <unique-path>.mov
ffmpeg -i <unique-path>.mov -vf "select='gt(scene,0.02)',showinfo" -f null - 2>&1
```

For repeated-flow memory investigation, warm the flow, sample a settled baseline, then sample after repeated batches such as 10x, 20x, and 30x with an idle control. Growth must remain material across batches and be attributable to the app-owned flow; one before/after increase or an unattributed nonzero `leaks` result is inconclusive.

```
leaks <pid>
footprint <pid>
```

## Wave composition

Before wave 1, ensure `APPMAP.md` contains at least launch/reset instructions, global navigation, known journeys, prerequisites, and test data. It may be source-derived initially. Runtime discoveries refine it.

Before wave 1 and after every worker, assert each simulator's environment baseline and restore it when it drifts: locale and language, appearance, content size, status-bar overrides, and any privacy grants a worker changed. Record the baseline commands in the map's launch/reset block. A worker that inherits a contaminated simulator produces false positives that look exactly like defects. The app's data container is part of the baseline: each finder starts from the map's documented reset to a named fixture state, not from whatever the previous worker left behind.

Assign one simulator-bound finder per free simulator, and name the one UDID it may drive. With the host protocol's two simulators:

- **SIM-A:** Track A tests and expands one highest-value mapped journey.
- **SIM-B:** Track B inspects one different journey's representative states and screen family for operability and consistency.
- **Off simulator:** one or two non-overlapping Track C auditors inspect different risk areas or source partitions.
- **Track D:** interleave only after a simulator frees up and a named journey or symptom warrants measurement.

Merge map and coverage deltas after the wave, then assign later waves from uncovered journeys, states, and applicable risks in the ledger's coverage section. Give fresh agents a short summary of prior findings, known dead ends, and their exact coverage gap. Never spawn agents merely to keep a simulator busy, but while a fixer or verifier occupies one simulator and the host's spawn condition holds, keep the free simulator on the next uncovered journey rather than idle.

Treat dispatch as atomic: build and inspect the complete prompt before launching the worker, then retain its task ID. If a running worker needs corrected context, message that exact task ID. Never launch a separate "correction" agent. If messaging is unavailable, stop the malformed worker and launch one replacement. Each finder prompt and the dispatch delta name the escalation tier of the assignment; above Tier 0 they also state the gate evidence (every core journey `core-covered`, `blocked`, or `unsupported`). After dispatching the wave, emit one compact dispatch delta and stop; do not repeat status narration while workers run.

## Escalation

A wave is dry for a track when it produces no new distinct-root-cause candidate, as defined by the host protocol. Coverage deltas still direct the next assignment but do not count as candidates. A dry pass on one JID does not exhaust Tier 0 while another journey in the ledger's coverage section remains `discovered` or `attempted`; assign the applicable Track A/B finder to that uncovered journey next. Advance beyond Tier 0 only after the core-journey frontier is `core-covered`, `blocked`, or `unsupported`. A newly discovered journey returns applicable work to Tier 0. After that gate, never repeat an escalation tier that came up dry.

Run one wave per tier, then escalate as the host protocol requires. Diversify categories when a wave has multiple finders, risk-rank them for the actual app, and explicitly skip irrelevant categories rather than running generic checks blindly.

### Track A ladder

**Tier 0 - functional depth:**

1. CRUD or equivalent lifecycle: create, reopen, edit/cancel, save, delete/undo, relaunch.
2. Validation and recovery: invalid to corrected input without losing unrelated work.
3. Cross-view consistency: list/detail, counts, search, filter, sort, and stale refresh behavior.
4. Interruption and concurrency: repeated actions, back/cancel during work, background/relaunch, stale responses.
5. Empty, populated, maximum, duplicate, partial, and failed states applicable to each core journey.

**Tier 1 - cheap applicable conditions:**

1. Permission denial for capabilities the app actually requests:
   ```
   xcrun simctl privacy <UDID> revoke <service> <bundle-id>
   xcrun simctl privacy <UDID> reset all <bundle-id>
   ```
2. Declared deep links, each cold and warm:
   ```
   xcrun simctl openurl <UDID> "<url>"
   ```
3. Declared push routes with sanitized payloads:
   ```
   xcrun simctl push <UDID> <bundle-id> <unique-payload>.json
   ```
4. Date and timezone boundaries where the app displays, groups, expires, or calculates dates:
   ```
   SIMCTL_CHILD_TZ=Pacific/Kiritimati xcrun simctl launch <UDID> <bundle-id>
   ```

Terminate before applying launch-only conditions. Restore and verify the baseline after each assignment.

**Tier 2 - changed environment:**

5. Locale and RTL when localization is supported; find navigation and meaning failures, not untranslated-copy inventory:
   ```
   xcrun simctl launch <UDID> <bundle-id> -AppleLanguages "(ar)" -AppleLocale "ar_SA"
   ```
6. Rotation, multitasking, role/session expiry, import/export, media interruption, or hardware-related states only when supported and automatable.

**Tier 3 - disruptive or manual:**

7. Offline, slow, or lossy networking. Simulator networking affects the host; obtain user approval first.
8. Memory warning, rotation when automation is unavailable, or physical-device-only capabilities. Mark unsupported coverage honestly.
9. iPad split view for universal apps.

**Tier 4 - accessibility-specific audit:**

10. VoiceOver meaning and focus order, labels/values/traits/actions, announcements, nonvisual discoverability, and accessibility-specific system settings. This is escalation-only unless the host requests it earlier. Grade defects with the same user-impact ladder.

### Track B ladder

**Tier 0 - representative state coverage:**

1. Compare equivalent controls and components across core journeys.
2. Capture validation, keyboard-up, loading, error, empty, success, destructive, and completed states where applicable.
3. Sample light/dark and smallest/largest supported target sizes on flagship states.
4. Exercise realistic long and boundary content using app-valid data.

**Tier 1 - applicable compatibility conditions:**

1. Pseudo-localized long strings for fixed-width and meaning failures:
   ```
   xcrun simctl launch <UDID> <bundle-id> -NSDoubleLocalizedStrings YES
   ```
   The flag doubles whole format strings, so a one-argument format shows leaked specifier fragments (`lld`, `@`). That is a property of the pseudo-locale, not a defect; only clipping, wrapping that changes meaning, and lost tap targets count.
2. Status-bar extremes where layout uses the top safe area:
   ```
   xcrun simctl status_bar <UDID> override --time "23:59" --batteryState charging --batteryLevel 4
   xcrun simctl status_bar <UDID> clear
   ```
3. Supported orientation and iPad size-class changes.

Terminate before launch-only conditions and restore the baseline after each assignment.

**Tier 2 - transient motion, symptom-driven:**

Record only interactions with a visible flash, identity swap, stale frame, jump, or ambiguous completion. Use scene detection to locate inspection windows, then judge explicit invariants; do not use scene-change counts as a defect threshold.

```
xcrun simctl io <UDID> recordVideo --codec h264 --force <unique-path>.mov
ffmpeg -ss <t> -t 1.2 -i <unique-path>.mov -vf "fps=30,scale=360:-1" <unique-dir>/f%02d.png
ffmpeg -i <unique-dir>/f%02d.png -frames:v 1 -vf tile=4x5 <unique-path>-sheet.png
```

**Tier 3 - accessibility-specific presentation audit:**

Dynamic Type extremes, Increase Contrast, Differentiate Without Color, Reduce Motion, and related settings run here unless requested earlier. A standards mismatch becomes a candidate only with category-appropriate evidence and severity based on impact.

```
xcrun simctl ui <UDID> content_size accessibility-extra-extra-extra-large
xcrun simctl ui <UDID> increase_contrast enabled
xcrun simctl ui <UDID> increase_contrast disabled
```

### Track C and Track D escalation

Track C runs once per non-overlapping source partition and re-runs when new map leads or post-fix code warrant it. Track D runs once per named journey and metric; re-run only after fixes touch the measured behavior or prior measurements are inconclusive for a correctable reason.

## When to hunt and when to stop

A round is one hunt against one ledger (the `round-N` of priors); a wave is one dispatch inside it. The host's target count is a ceiling, not a quota.

- **When to hunt:** start a round for a release candidate or a feature that just landed, not as a repeat on an app with no release date. Before wave 1, read the last two rounds' verified severities from `HUNT.md` or their ledgers. If every verified fix in both was S3/S4, tell the user once that shipping beats another round, and repeat that recommendation in the final report.
- **Stop rule:** once the Tier 0 gate in "Escalation" is met, the first wave that confirms nothing above S3 ends finding. Dispatch no more finders, carry CONFIRMED and FIXED-UNVERIFIED issues through fix and verification, then file the final report. Do not climb further tiers to reach the count.
- **Attempt cap:** the host's two-rejection rule is final for the round. A fix that fails verification twice becomes `BLOCKED`, with each attempt's approach and failure in its row, and goes to the priors' Fixers tar pits at wrap-up. No third or fourth attempt this round, on any model.
- **Ledger hygiene:** ledger, coverage, builds, and map edits stay in the working tree and land in one commit when the round closes, with the `HUNT.md` append. Never commit per restamp, note, or status change; fixers stage only their own files.
- **Final report:** state what ended the round (count reached, turn limit, or stop rule), and append `round-N verified: S1 <n>, S2 <n>, S3 <n>, S4 <n>` to `HUNT.md` so the next round can apply the when-to-hunt check.

## Blind verification addendum

Verifiers receive the defect description, exact repro, expected behavior, target build path, `APPMAP.md`, assigned UDID, and required environment or measurement procedure. Write the expected behavior from the original finding's expected basis, never from the fixer's report: describing what the fix now shows tells the verifier what to look for. Name builds neutrally; the prompt carries no fix hash, fix approach, or source type name that the map does not already use, and any coordinates in it come from the verifier's own device. Explicitly forbid reading source code, diffs, git history, commit messages, or implementation notes. All verdict evidence comes from driving the installed build.

Before replay, record the tested app path/build identity, target, simulator UDID, OS, fixture/account, and relevant condition. Restore the original preconditions. Use a fresh mid-tier-or-better simulator agent and the budget block's verifier cap.

Allowed verdicts align with the host ledger:

- `VERIFIED`: the original defect is absent and the expected user outcome is observed.
- `REJECTED`: the original defect or an equivalent harmful behavior remains.
- `BLOCKED`: a prerequisite or repro path cannot be completed within the cap.

These are verifier verdicts. A verifier's `REJECTED` means the attempted fix failed and the confirmed issue returns for another fix; it is distinct from rejecting an unconfirmed candidate as intentional, duplicate, unsupported, or false positive. If the path completes but evidence cannot distinguish fixed from unfixed, return `BLOCKED: inconclusive evidence` and explain what is missing. For `VERIFIED`, rerun the exact repro, describe the observed state at the former defect location, and check one named adjacent invariant such as cancellation, repeated action, persistence, or the equivalent control elsewhere.

- **Device and condition:** verify on the device class where the defect was found, and on the smallest supported width for any layout or truncation fix. If that simulator is busy, wait for it rather than verify elsewhere.
- **Motion claims:** a fix described as removing a jump, jitter, flash, or lag needs a short video or contact sheet; still screenshots cannot support it.
- **Residuals:** when the fixer reported an OS-availability gate, an untested device, or a capability-reducing change, a passing verdict is recorded as `VERIFIED` with the residual stated in the row, and the uncovered case is added to that journey's `next:` in coverage.
- **Track B fixes:** capture fresh evidence appropriate to the claim and describe the relevant pixels and behavior. "Looks fixed" is not a verdict.
- **Track D fixes:** repeat the same measurement under comparable conditions and quote all runs, median, and worst case against the original values.
- **Timing pitfall:** the harness-artifact rule in "Candidate and evidence rules" applies to verifiers too. If using `idb`, set its assigned UDID, warm it with a throwaway describe command, bracket the tap and recording start with wall-clock timestamps, and calculate the tap offset. If timing worsens unexpectedly, investigate measurement error and host load before claiming regression.

## Notes

- Keep environment changes scoped to the assigned UDID and restore baseline settings after each worker.
- Accessibility-specific auditing is late by design for this workflow; observable usability failures remain core regardless of which users they affect.
- Escalation order reflects likely defect yield and execution cost, not possible severity.
- Re-check commands against the installed Xcode and simulator runtime before relying on them.
