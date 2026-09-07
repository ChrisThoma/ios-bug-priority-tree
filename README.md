# ios-bug-priority-tree

An agent skill for [Claude Code](https://claude.com/claude-code) and [Codex](https://developers.openai.com/codex/cli) that guides a long, simulator-driven iOS bug hunt around user journeys, UI functionality, understandable state, consistency, recovery, edge cases, and demonstrated user impact.

Point a few agents at an app and tell them to "find bugs" and you get one wave of easy findings, then diminishing returns, then a lot of tokens spent re-walking the same happy paths. This skill gives an external goal/orchestration loop a testing method: map meaningful user goals, verify state transitions instead of counting visited screens, record coverage gaps, and escalate into relevant conditions only after normal journeys have real depth.

Severity runs S1 to S4: S1 is catastrophic harm, S2 is task failure or a harmful result, S3 is a material UX failure, and S4 is pure polish. Severity follows the demonstrated user consequence rather than the audit category. S1 through S3 count toward the goal; at most two S4 findings may count so agents cannot satisfy a hunt with alignment nits.

## Install

With the skills CLI (works for Claude Code, Cursor, Codex, and others):

```bash
npx skills add ChrisThoma/ios-bug-priority-tree
```

Or clone the repo and copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/ChrisThoma/ios-bug-priority-tree.git
cp -r ios-bug-priority-tree/skills/ios-bug-priority-tree ~/.claude/skills/
```

For Codex, copy into `~/.codex/skills/` instead — both hosts read the same `SKILL.md`, only the directory differs. Or for a single project, copy into `.claude/skills/` (or `.codex/skills/`) inside the repo.

## Use

Invoke it directly with `/ios-bug-priority-tree` in Claude Code or `$ios-bug-priority-tree` in Codex. It is a policy and testing-method companion to a "find N distinct issues" goal loop, not an executable test harness. The host loop owns builds, turns, commits, issue status, and scheduling. Load the skill before the first finder wave so its Track A-D interface, severity ladder, app-map guidance, and escalation rules apply from the beginning. Each finder receives only its own track section plus the shared budget and app map.

## What it does

1. **Map** the app in `APPMAP.md` as user goals, entry paths, prerequisites, outcomes, meaningful state branches, test data, and coverage gaps. Existing screen-only maps are migrated before reuse. Screens remain navigation landmarks; they are not the unit of coverage.
2. **Test** through four stable tracks. Track A completes journeys and verifies data and persistence. Track B inspects visual-functional state and consistency during real journeys. Track C connects source risks and hidden routes to user impact. Track D measures named, high-value journeys when a symptom warrants it.
3. **Probe** state transitions. A normal journey check follows baseline, action, immediate feedback, settled outcome, cross-view consistency, and return or relaunch, then applies relevant validation, cancel, retry, duplicate-action, interruption, and recovery cases.
4. **Escalate** from uncovered normal states into applicable permissions, routes, dates, locales, device conditions, motion analysis, and disruptive tests. Repository priors specialize product facts and known risks without silently replacing journey-first ordering. Accessibility-specific auditing is a late tier unless explicitly requested as a current-hunt objective; using the accessibility tree to drive the simulator is only automation plumbing.
5. **Demand impact** before counting a finding. A checklist match is not automatically S3, ambiguous product intent is a product question, and ordinary source smells remain leads until runtime behavior confirms them.
6. **Budget** context with fresh agents, assigned UDIDs, bounded evidence, map deltas, and concise reports.
7. **Verify** blind. Verifiers receive the defect, repro, app map, build path, and environment, but not source, diffs, history, commit messages, or implementation notes. They repeat the user journey and one named adjacent invariant.

See [SKILL.md](skills/ios-bug-priority-tree/SKILL.md) for the full tree.

## Requirements

Baseline requirements are Xcode command line tools (`xcrun simctl`), `sips`, and a server that can drive a specifically assigned simulator. The quoted tool names are [ios-simulator-mcp](https://github.com/joshuayoes/ios-simulator-mcp)'s; substitute your own server's equivalents. `ffmpeg`, `idb`, `leaks`, and `footprint` are conditional requirements for motion, timing, or memory escalation rather than prerequisites for every hunt.

On Codex, subagents need the `multi_agent` feature, which is stable and on by default (`codex features list | grep multi_agent`). On Claude Code, no extra setup is needed. The `agents/openai.yaml` in the skill folder supplies Codex's display name and default prompt; Claude Code ignores it.

Use disposable simulators, sandbox accounts, and synthetic data. Do not aim destructive or repeated-action probes at production services. Re-check commands against the installed Xcode and simulator runtime before relying on them.

## License

[MIT](LICENSE)
