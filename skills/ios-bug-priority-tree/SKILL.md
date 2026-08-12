---
name: ios-bug-priority-tree
description: Finder tracks, severity ladder, and escalation tiers for an iOS bug-hunt /goal loop. Load at the START of the hunt — it defines what counts as an issue (S1–S4), the four finder modalities (UI-flow, visual/design, code audit, performance) composed into every wave, per-track escalation when a wave runs dry, and how verifiers must check visual and performance fixes.
---

# iOS bug-hunt: tracks, severity, and escalation

Companion to a "find N distinct issues" iOS `/goal` protocol. Load this at hunt start, before the first finder wave — track assignment and the severity ladder apply from wave one. The escalation tiers apply later, when a track's wave runs dry.

## Agent budgets — include this block verbatim in every subagent prompt

> **Budget.** Context cost grows with every turn you take — each turn re-reads your entire history, so a 300-turn session costs ~100× a 30-turn one, not 10×. Work within these caps and return early rather than push past them:
> - **Finders:** return after **~30 simulator interactions or 3 candidate issues**, whichever comes first. If you cannot reach an assigned screen within 10 actions, record it as unreachable and move to the next area — do not keep retrying.
> - **Verifiers:** the repro must complete in **≤15 simulator actions**. If it doesn't, return `BLOCKED: could not complete repro path` with what you tried — that is a valid verdict, not a failure.
> - **Navigation:** use `ui_find_element` (or describe a point) to locate controls. A full `ui_describe_all` dump is allowed at most **once per new screen** — never after every tap.
> - **Screenshots:** the screenshot tool writes a file; **downscale before viewing** (`sips -Z 750 <file>`) and **Read each image at most once** — a full-resolution screenshot re-Read into context is paid on every later turn. Take screenshots only when your track's method requires them or to capture final evidence; never as a general "where am I" check (use the tree for that).
> - **One screenshot per verdict/finding** is the evidence ceiling for Tracks A and D and for all verifiers.
> - **Report format:** your final report is a structured list — per finding/verdict: repro steps, expected vs actual, evidence, severity guess. **≤25 lines total.** Never narrate your process, dead ends, or turn-by-turn history; one line each for dead ends worth flagging. The orchestrator pays for every line you return on every one of its later turns.

## Severity ladder — what counts as an issue

Replaces any binary "cosmetic / not cosmetic" test. Every candidate gets a severity at CONFIRM time:

- **S1** — crash, data loss, security exposure.
- **S2** — broken flow, wrong data, lost input, unhandled error surfaced to the user.
- **S3** — misleading state / UX failure: the user can't tell what happened or is led to a wrong conclusion. Includes: missing pressed/loading/disabled state on an async or destructive control; truncation or wrapping that changes a label's meaning; contrast that hides information (error text invisible against its background); a tap target too small to hit reliably; keyboard covering the field being edited; an empty state indistinguishable from a loading or error state.
- **S4** — pure polish: alignment, spacing, non-semantic color choices, awkward-but-unambiguous layout.

**Rules:** S1–S3 count toward the goal. S4 findings are logged in the ledger but at most 2 of the goal count may be S4 — finders must not farm polish. Distinct-root-cause dedup applies across all severities. A defect found by any track goes through the same CONFIRMED → FIX → VERIFY pipeline.

## Finder tracks — four modalities, all live from wave one

Hand each finder only its own track's section and assigned commands — never this whole file. The tree lives in the orchestrator's context only.

**Evidence format per track** (overrides any blanket "one screenshot per issue" rule): Tracks A and B attach one screenshot per issue; Track C cites file:line plus the user-visible repro (no screenshot — it has no simulator); Track D quotes its measurements (stall seconds, before/after memory numbers). Track C additionally returns its leads (findings without a user-visible repro yet) in a separate list.

### Track A — UI-flow (simulator, accessibility-tree driving)

Walk screens, forms, nav paths; check data correctness, error states, lost input. Navigate via the accessibility tree; screenshots only as evidence. Baked-in free checks — include in every Track A prompt from wave one:

- **Rapid double-tap** — "on any submit / purchase / like / destructive control you encounter, tap it twice in quick succession and check for duplicate actions or crashes."
- **Force-quit & relaunch mid-flow** — "once per assigned area, kill and relaunch the app mid-flow and check that state restores sanely rather than being lost or corrupted."
  ```
  xcrun simctl terminate <UDID> <bundle-id>
  xcrun simctl launch <UDID> <bundle-id>
  ```
- **Empty vs. huge data** — where the finder can control content, try zero items and absurdly many / absurdly long input; check for indistinguishable empty states, broken layout, lost input on limit.
- **A sudden return to springboard IS a crash** — never dismiss it as a "transition." Before writing it off as unreproducible, check for a fresh report in `~/Library/Logs/DiagnosticReports/` (host path — simulator crashes land there), and replay the FULL sequence that got you there (if it happened while deleting, delete all the way down again).

### Track B — Visual/design audit (simulator, screenshot driving)

**Screenshots ARE the method for this track** — the explicit exception to the a11y-tree rule and to the one-screenshot evidence ceiling (the budget block's downscale-and-Read-once rule still applies: `sips -Z 750` before viewing, never re-Read). The finder navigates by tree but screenshots every distinct screen and *reads the pixels* against this rubric:

- Tap targets ≥ 44pt; controls not overlapping or clipped.
- Contrast: every piece of information legible against its actual background — especially error/status banners, placeholder text, disabled states.
- Interactive feedback: tap an async/destructive control, then screenshot at ~0.3s and ~1s while the work is in flight. If both screenshots show the UI looking idle (control still enabled, no spinner/disabled/progress state) while the operation has not completed, that IS an S3 finding — do not downgrade it to "inconclusive" for lack of a visible spinner; the absence is the defect. A momentary tint flash is not feedback for a multi-second operation.
- Text: wrapping and truncation with real long content; labels that ellipsize into ambiguity or wrap into misalignment.
- Safe area: content under notch/home-indicator; keyboard avoidance (screenshot with keyboard up on every input screen).
- Empty states: distinguishable from loading and from error.

Run the rubric in **light and dark mode** (dark is a first-pass condition here, not an escalation):
```
xcrun simctl ui <UDID> appearance dark|light
```
Severity per the ladder: hides/changes information → S3; pure aesthetics → S4 (log, don't chase).

### Track C — Static code audit (no simulator — free parallelism)

Reads app source; needs no simulator, so it runs alongside A/B from wave one at zero sim cost. Targets: logic errors (off-by-one, date/timezone/locale math, boundary conditions), unhandled errors and empty catch blocks, races beyond UI double-tap (shared mutable state, unawaited tasks), state-management smells — especially one logical change written across two state systems (`@Observable` store mutation + local `@State` reset), force-unwraps and `try!` on user-reachable paths, ignored return values on save/commit calls.

**Lead rule:** a code finding becomes a CANDIDATE only once expressed as a user-visible repro (which screen, what steps, what the user sees). A finding the code-auditor can't script into a repro is a **lead**: the orchestrator hands it to a Track A finder ("try to make this visible: <lead>") in the next wave. Leads are free hypotheses, not ledger entries.

### Track D — Performance probe (simulator + host tools)

Simulator apps are host processes, so host tools attach directly. Thresholds — below these, it's not an issue:

- **Visible hang > 1s** during any interaction; **cold launch > 2s** to first content.
  ```
  xcrun simctl io <UDID> recordVideo --codec h264 --force out.mov   # stop with kill -INT
  ffmpeg -i out.mov -vf "select='gt(scene,0.02)',showinfo" -f null - 2>&1 | grep pts_time
  ```
  Record the cold launch or the suspect interaction; the pts_time gaps give you the stall duration. Quote seconds in the finding.
- **Monotonic memory growth** across a repeated flow (open/close the same screen 10×):
  ```
  leaks <pid>          # leak count and bytes
  footprint <pid>      # phys_footprint / peak
  ```
  Get the pid from `xcrun simctl launch` output. Growth that never plateaus, or nonzero leaks tied to the repeated flow, is the finding — quote before/after numbers.

## Wave composition

With 2 simulators: **wave 1 = Track A on SIM-A, Track B on SIM-B, plus 1–2 Track C agents off-simulator.** Track D interleaves when a simulator frees up (verify phases are a natural gap). Never spawn an agent just to keep a simulator busy. If an SE-class device type is available, prefer it for one of the sims — smallest-screen layout is free coverage.

**Every wave and every escalation tier gets a fresh agent.** Never continue a finder from a previous wave via SendMessage, and never run multiple ladder tiers inside one agent session — a long-lived agent re-pays its entire accumulated history (screenshots included) on every turn. Hand the fresh agent a ≤10-line summary of what prior waves found and which areas came up dry.

## Escalation — per track, when a wave runs dry

- **One dry wave escalates.** Zero new distinct-root-cause candidates from a track's wave → move that track up its ladder. Never repeat a tier that just came up dry.
- **One wave per tier.** Then escalate regardless.
- **Diversify within the wave**: assign each finder a *different* category from the tier.
- **Don't skip tiers**; skip a *category* only if it can't apply.

### Track A ladder

**Tier 1 — cheap, simctl-scriptable:**

1. **Permission denial** — revoke, then exercise the flow that needs it:
   ```
   xcrun simctl privacy <UDID> revoke <service> <bundle-id>
   ```
   Services: `location`, `location-always`, `photos`, `photos-add`, `contacts`, `contacts-limited`, `calendar`, `reminders`, `microphone`, `media-library`, `siri`. Reset: `simctl privacy <UDID> reset all`.
2. **Deep links / URL schemes** — each link cold (terminated) and warm (mid-flow elsewhere):
   ```
   xcrun simctl openurl <UDID> "<url>"
   ```
3. **Push notification routing:**
   ```
   xcrun simctl push <UDID> <bundle-id> payload.json   # top-level aps key, ≤4096 bytes
   ```
4. **Date/timezone edges** — relaunch under a hostile zone and check every date the UI shows:
   ```
   SIMCTL_CHILD_TZ=Pacific/Kiritimati xcrun simctl launch <UDID> <bundle-id>
   ```

**Tier 2 — full re-walk of 2–3 key flows under changed conditions** (costs a normal finder pass):

5. **Locale / RTL** — mirrored-layout navigation breakage, not just untranslated strings:
   ```
   xcrun simctl launch <UDID> <bundle-id> -AppleLanguages "(ar)" -AppleLocale "ar_SA"
   ```

**Tier 3 — expensive, host-disruptive, or manual — only if still short:**

6. **Network fault injection** (offline/slow/lossy). No simctl API — the Simulator shares the host stack; toggling host Wi-Fi or Network Link Conditioner affects the orchestrator too. Confirm with the user first; keep it brief.
7. **Memory pressure** — manual only: Simulator app ▸ Device ▸ Trigger Memory Warning.
8. **Rotation / landscape** — no simctl command; AppleScript keystrokes to the Simulator app proved unreliable on this machine. Manual (⌘→ in Simulator) or skip for portrait-locked apps.
9. **Accessibility as a defect target** — a pass checking that accessibilityLabel/traits/hints match visible content, not just using the tree to navigate.
10. **iPad split-view / multitasking** — universal apps only.

### Track B ladder

**Tier 1 — re-run the rubric under one changed condition each:**

1. **Dynamic Type extremes** — critical text truncated past readability, controls pushed off-screen:
   ```
   xcrun simctl ui <UDID> content_size accessibility-extra-extra-extra-large
   ```
2. **Pseudo-localization long strings** — every label doubled; finds fixed-width assumptions without needing translations:
   ```
   xcrun simctl launch <UDID> <bundle-id> -NSDoubleLocalizedStrings YES
   ```
3. **Increase Contrast mode** — information that vanishes when the system remaps colors:
   ```
   xcrun simctl ui <UDID> increase_contrast enabled|disabled
   ```
4. **Fake status bar** — layouts that assume a short clock or full signal:
   ```
   xcrun simctl status_bar <UDID> override --time "23:59" --batteryState charging --batteryLevel 4
   xcrun simctl status_bar <UDID> clear
   ```

**Tier 2 — transient motion (record & frame-step).** Screenshots and the a11y tree only see settled end states; every other category is blind to what happens *during* an animation. Record each flagship animated interaction, then frame-step:
```
xcrun simctl io <UDID> recordVideo --codec h264 --force out.mov   # stop with kill -INT
ffmpeg -i out.mov -vf "select='gt(scene,0.02)',showinfo" -f null - 2>&1 | grep pts_time
ffmpeg -ss <t> -t 1.2 -i out.mov -vf "fps=30,scale=360:-1" f%02d.png && ffmpeg -i f%02d.png -frames:v 1 -vf tile=4x5 sheet.png
```
Look for: an element vanishing for a few frames, content flashing wrong data mid-transition, layout jumping after the animation "ends." 1–2 scene-change hits is clean; 4+ within half a second is a candidate — tile that window and read it. Cost control: 2–3 flagship interactions only, triage with the scene filter, never frame-step an unflagged recording. Aim first where the code smells: animation state applied by slot/index instead of identity (`index == 0 ? offset : .zero`), and completion blocks writing one logical change across two state systems — the frame between the two writes is the bug.

### Track C / D

Track C runs once; it re-runs only when new leads or new code (post-fix) exist. Track D runs once per distinct screen-set; re-run only after fixes touch the measured flows.

## Verification addendum — visual and perf fixes

Verifiers never see the fix or diff, only the defect description + repro — and their prompt must explicitly FORBID reading the app's source code, diffs, or git history (otherwise they go find the fix themselves and rubber-stamp it, which happened in testing). Verifiers that drive a simulator run on **sonnet, not haiku** — in testing a haiku verifier flailed for 326 turns and 35 screenshots on a 5-step repro; fewer turns beats cheaper turns. Verifier prompts include the budget block's 15-action cap. All verifier evidence comes from driving the installed build. A verdict that cites source code instead of runtime observation is inadmissible: reject it and spawn a new verifier. Two track-specific requirements:

- **Track B fixes:** the verifier re-runs the scenario, takes a fresh screenshot, and its verdict must describe what the screenshot now shows at the defect's location (e.g., "the error banner is now white-on-red and legible"). "Looks fixed" without citing pixel-level content is not a verdict — send it back.
- **Track D fixes:** the verifier re-runs the same measurement (same recordVideo/ffmpeg or leaks/footprint procedure, included in its prompt) and quotes before/after numbers against the threshold.
- **Timing pitfall (finders and verifiers both):** never *assume* when a tap happened — measure it. The first `idb` command in a fresh shell spins up a companion and can take seconds, so "I tapped ~1.5s after recording started" can be off by several seconds and manufactures a phantom hang. Procedure: warm idb first with a throwaway command (`idb ui describe-all` and discard it), bracket the measured tap with wall-clock timestamps (`python3 -c 'import time; print(time.time())'` before and after), record the recording-start wall-clock the same way, and compute the tap's pts offset from those — then stall = first scene-change pts − tap pts. If a measured stall exceeds the finder's original defect measurement, treat your own timing as the prime suspect before blaming a regression.

## Notes

- Tiers are ordered by cost-to-run vs. bug-yield, not by severity of what they might find.
- All commands verified against this machine's Xcode/simctl on 2026-08-08 (`status_bar`, `leaks`/`footprint` on a simulator PID, `SIMCTL_CHILD_TZ`, `-NSDoubleLocalizedStrings`, `increase_contrast`, recordVideo+ffmpeg). Re-check if the Xcode version differs.
