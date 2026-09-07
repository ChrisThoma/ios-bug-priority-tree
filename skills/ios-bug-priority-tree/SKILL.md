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

- Supported target, OS, orientations, app capabilities, account roles, and integrations.
- Test account, backend, fixtures, reset method, and whether destructive actions are safe.
- Product promises from requirements, tests, visible UI copy, and established behavior elsewhere in the app.
- Known limitations, unsupported conditions, and environment constraints.

Use sandbox accounts and synthetic data. Do not make real purchases, mutate production data, expose credentials or personal data in evidence, or perform irreversible external actions without explicit authorization. Host-wide network changes and other disruptive manual conditions require user approval and a restoration plan.

A finding's expected behavior must be grounded in a visible UI promise, documented requirement, data invariant, established equivalent behavior, or strong iOS convention. If the expected behavior is genuinely ambiguous, report a `PRODUCT QUESTION` lead rather than a candidate. Unreachable functionality is a coverage gap, not a defect, unless its entry point should be available to the tested user.

## APPMAP.md: journey and coverage map

Create `APPMAP.md` before wave 1 from bounded source inspection, app metadata, launch behavior, and supplied product information; runtime finders then return concise deltas when they discover new paths or prerequisites. If a map already exists, validate and migrate it to the journey schema below before reuse. Do not treat a screen list as a current coverage map. Preserve still-valid launch, reset, capability, and intentional-behavior facts while removing stale guesses. The orchestrator is the single writer.

Keep `APPMAP.md` compact and include it verbatim in every subagent prompt. It is the coverage map, not a second issue ledger. `ISSUES.md`, when supplied by the host protocol, remains the authority for defect status.

For each meaningful journey, record:

```
JID | user goal | entry path | prerequisites/test data | expected outcome
states/branches: success, validation, cancel, loading, empty, error, retry, completed (applicable only)
coverage: discovered | attempted | core-covered | blocked | unsupported
tested: conditions relevant to this journey
next: highest-value untested state or branch
```

Also record exact launch/reset commands, stable screen names, global navigation, test-data sentinels, capabilities, and blocked entry points. A journey is a user goal or meaningful task, not a screen. Visiting a screen does not cover its actions or states. A journey is `core-covered` only after its principal outcome and applicable state, cancellation, error, and persistence behavior have been observed.

Finder reports include only `APPMAP.md` deltas: new or corrected journeys, coverage changes, blockers, and the next useful probe. Do not return a rewritten map. Broad discovery is exhausted only when the known frontier is empty and fresh runtime passes plus bounded source inspection reveal no new meaningful journey or high-risk branch.

## Repository priors and precedence

Use `HUNT.md` or equivalent repository priors to specialize the skill, not to silently replace its purpose. Product facts, known intentional behavior, safety constraints, capability limits, build traps, proven dead ends, and defect-specific verifier conditions take precedence over generic skill assumptions.

Historical bug yield is assignment evidence, not a standing waiver of journey-first ordering. Track weights, wave-one mandates, or instructions that move accessibility-specific checks ahead of core UX/functionality apply only when they are explicitly marked as a current-hunt objective. Otherwise retain them as leads for the applicable escalation tier. When a prior conflicts with the current host objective, the current objective wins; record the conflict rather than combining contradictory instructions.

## Agent budgets - include this block verbatim in every subagent prompt

> **Budget.** Context cost grows with every turn you take. Work within these caps and return early rather than push past them:
> - **Finders:** return after **about 30 simulator interactions or 3 candidate issues**, whichever comes first. Prioritize completing an assigned journey over counting screens. If setup consumes most of the budget, return the exact checkpoint and remaining probes. If you cannot reach an assigned path within 10 actions beyond its documented setup, record it as blocked and move on.
> - **Verifiers:** the defect repro itself must complete in **15 simulator actions or fewer**, excluding documented fixture setup. If it does not, return `BLOCKED: could not complete repro path` with what you tried.
> - **Navigation:** use the assigned UDID and the simulator server's find-element call (`ui_find_element`, or describe a point) to locate controls. A full tree dump (`ui_describe_all`) is allowed at most **once per new screen state**, never after every tap.
> - **Screenshots:** downscale a viewing copy before inspection (`sips -Z 750 <copy>`), retain the original evidence, and view each image at most once. Take screenshots when the track's method requires them or to capture evidence, not as a general navigation aid.
> - **Evidence:** return one primary artifact per finding. A bounded before/after pair or short video/contact sheet is allowed when a single image cannot prove the claim.
> - **Report:** return a structured list with at most 3 candidates. For each: journey ID, preconditions, exact repro, expected basis, expected vs actual, user impact, reproduction count when practical, evidence, severity guess, and suspected duplicate. Add compact `APPMAP.md` deltas and worthwhile leads. Keep the report at **25 lines or fewer** and do not narrate process or dead ends.

## Severity ladder

Assign severity from demonstrated user impact, not from the finder track or checklist category:

- **S1 - catastrophic:** crash on a consequential or normally reachable path, unrecoverable data loss, security/privacy exposure, or a severe unintended irreversible action.
- **S2 - task failure or harmful result:** a meaningful user goal is blocked; committed or displayed data is wrong; input is lost; an action produces duplicate or unintended effects; recovery requires abandoning the journey or repairing data outside its normal flow.
- **S3 - material UX failure:** the task remains possible, but the app misleads the user, obscures necessary state, behaves inconsistently enough to cause a wrong action, or cannot be operated reliably. Examples include indistinguishable loading/error/empty states, absent feedback that leaves a consequential action ambiguous, keyboard obstruction, meaningful truncation, recoverable lost progress, or an unreliable control.
- **S4 - polish:** visible inconsistency or aesthetic degradation that does not obscure meaning, change behavior, mislead the user, or reduce reliable operation.

Record reproduction confidence, journey importance, recoverability, and affected conditions separately from severity. A rubric match is not automatically S3: state the actual user consequence. The host goal may count S1-S3 plus at most two S4 findings. Deduplicate by demonstrated behavior and triggering condition first; merge by root cause only when evidence supports it.

## Candidate and evidence rules

Track A, B, and D candidates require runtime evidence from the installed build. Reproduce from a known baseline twice when practical; record results such as `2/2` or `1/3`. One clear reproduction may be sufficient for destructive S1 behavior that is unsafe to repeat. Tool failure, environment contamination, unsupported capabilities, and uncertain product intent are not candidates.

Use evidence appropriate to the claim:

- Stable visual or state defect: screenshot.
- Lost or inconsistent state: bounded before/after pair with the same sentinel data.
- Transient interaction defect: short video or contact sheet with timestamps.
- Crash: full triggering sequence and crash-report reference; a return to Springboard is a crash lead until process exit or a crash report supports it.
- Performance: commands, conditions, all measurements, and summary statistic.
- Source lead: file and line or symbol plus a proposed user-visible repro; no screenshot required.

A Track C finding may be a candidate without driving a simulator only when it includes file/line evidence and a precise user-visible repro that a simulator finder can drive with existing launch arguments. Otherwise it is a lead. Security, privacy, silent data-integrity, or background-behavior risks that cannot meet the user-visible gate remain leads with category-appropriate supporting evidence.

## Finder tracks

Hand each finder the budget block, `APPMAP.md`, host priors that apply to it, and only its own track section. Every track reports user impact and map deltas using the common rules above.

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

Assign one simulator-bound finder per free simulator, and name the one UDID it may drive. With the host protocol's two simulators:

- **SIM-A:** Track A tests and expands the highest-value mapped journeys.
- **SIM-B:** Track B independently maps screen families and state cues while inspecting representative journeys for operability and consistency.
- **Off simulator:** one or two non-overlapping Track C auditors inspect different risk areas or source partitions.
- **Track D:** interleave only after a simulator frees up and a named journey or symptom warrants measurement.

Merge map deltas after the wave, then assign later waves from uncovered journeys, states, and applicable risks. Give fresh agents a short summary of prior findings, known dead ends, and their exact coverage gap. Never spawn agents merely to keep a simulator busy.

## Escalation

A wave is dry for a track when it produces no new distinct-root-cause candidate, as defined by the host protocol. Coverage deltas still direct the next assignment but do not count as candidates. Never repeat a tier that just came up dry. If a new journey appears, include its core testing in the next applicable assignment without undoing the track's required escalation.

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

## Blind verification addendum

Verifiers receive the defect description, exact repro, expected behavior, target build path, `APPMAP.md`, assigned UDID, and required environment or measurement procedure. Explicitly forbid reading source code, diffs, git history, commit messages, or implementation notes. All verdict evidence comes from driving the installed build.

Before replay, record the tested app path/build identity, target, simulator UDID, OS, fixture/account, and relevant condition. Restore the original preconditions. Use a fresh mid-tier-or-better simulator agent and the budget block's verifier cap.

Allowed verdicts align with the host ledger:

- `VERIFIED`: the original defect is absent and the expected user outcome is observed.
- `REJECTED`: the original defect or an equivalent harmful behavior remains.
- `BLOCKED`: a prerequisite or repro path cannot be completed within the cap.

These are verifier verdicts. A verifier's `REJECTED` means the attempted fix failed and the confirmed issue returns for another fix; it is distinct from rejecting an unconfirmed candidate as intentional, duplicate, unsupported, or false positive. If the path completes but evidence cannot distinguish fixed from unfixed, return `BLOCKED: inconclusive evidence` and explain what is missing. For `VERIFIED`, rerun the exact repro, describe the observed state at the former defect location, and check one named adjacent invariant such as cancellation, repeated action, persistence, or the equivalent control elsewhere.

- **Track B fixes:** capture fresh evidence appropriate to the claim and describe the relevant pixels and behavior. "Looks fixed" is not a verdict.
- **Track D fixes:** repeat the same measurement under comparable conditions and quote all runs, median, and worst case against the original values.
- **Timing pitfall:** never estimate when a tap occurred. If using `idb`, set its assigned UDID, warm it with a throwaway describe command, bracket the tap and recording start with wall-clock timestamps, and calculate the tap offset. If timing worsens unexpectedly, investigate measurement error and host load before claiming regression.

## Notes

- Keep environment changes scoped to the assigned UDID and restore baseline settings after each worker.
- Accessibility-specific auditing is late by design for this workflow; observable usability failures remain core regardless of which users they affect.
- Escalation order reflects likely defect yield and execution cost, not possible severity.
- Re-check commands against the installed Xcode and simulator runtime before relying on them.
