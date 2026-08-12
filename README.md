# ios-bug-priority-tree

A skill for running a long iOS bug hunt with subagents against the iOS Simulator. Works in both [Claude Code](https://claude.com/claude-code) and [Codex](https://developers.openai.com/codex/cli).

The problem it solves: point a few agents at an app and tell them to "find bugs," and you get a first wave of easy findings, then diminishing returns, then a lot of tokens spent re-walking the same happy paths. This skill gives the orchestrator a plan for what to do after the easy bugs are gone, and a shared definition of what counts as a bug so agents stop arguing with themselves about whether something is "just cosmetic."

## What's in it

**A severity ladder (S1 to S4).** S1 is a crash or data loss, S2 is a broken flow or wrong data, S3 is misleading state (the user cannot tell what happened), S4 is pure polish. S1 to S3 count toward the goal; S4 is capped at 2 so finders don't farm alignment nits.

**Four finder tracks that all run from wave one:**

- Track A drives the app through the accessibility tree, looking for broken flows, lost input, and bad error states.
- Track B screenshots every screen and reads the pixels against a rubric (tap targets, contrast, truncation, keyboard avoidance, empty states), in light and dark mode.
- Track C reads the source with no simulator attached, so it runs in parallel for free. Its findings only become candidates once someone turns them into a user-visible repro.
- Track D measures: hangs over 1s, cold launch over 2s, memory that grows and never plateaus.

**Escalation ladders per track.** When a track's wave comes up empty, that track moves up its ladder instead of repeating itself. Track A escalates to permission denial, deep links, push routing, hostile timezones, then RTL, then the expensive manual stuff. Track B escalates to Dynamic Type extremes, pseudo-localization, increased contrast, fake status bars, then frame-stepping recorded animations to catch bugs that only exist mid-transition.

**Budget rules.** Each agent turn re-reads the whole conversation, so a 300-turn agent costs far more than ten 30-turn agents. The skill caps finder turns, forbids screenshot re-reads, requires a fresh agent per wave, and holds reports to 25 lines.

**Verification rules.** Verifiers are barred from reading the source, the diff, or git history, because a verifier that finds the fix will rubber-stamp it. They have to drive the installed build and describe what they actually saw.

Most of this came out of watching runs go wrong: a verifier that spent 326 turns on a 5-step repro, agents dismissing a return-to-springboard as "a transition," phantom hangs caused by mistiming a tap rather than by the app.

## Install

Both agents read the same `SKILL.md`; only the directory differs. Global installs are shown here, but per-project `.claude/skills/` and `.codex/skills/` work the same way.

One line per host, no clone and no git history, using [degit](https://github.com/Rich-Harris/degit) to pull just the skill folder:

```bash
# Claude Code
npx -y degit ChrisThoma/ios-bug-priority-tree/skills/ios-bug-priority-tree ~/.claude/skills/ios-bug-priority-tree

# Codex
npx -y degit ChrisThoma/ios-bug-priority-tree/skills/ios-bug-priority-tree ~/.codex/skills/ios-bug-priority-tree
```

Add `--force` after `degit` to overwrite an existing copy when you update. For a project-local install, point it at `.claude/skills/ios-bug-priority-tree` or `.codex/skills/ios-bug-priority-tree` instead.

Or clone and copy, if you would rather read the whole repo first:

```bash
git clone https://github.com/ChrisThoma/ios-bug-priority-tree.git
cp -r ios-bug-priority-tree/skills/ios-bug-priority-tree ~/.claude/skills/   # Claude Code
cp -r ios-bug-priority-tree/skills/ios-bug-priority-tree ~/.codex/skills/    # Codex
```

It is picked up on the next session. Invoke it directly with `/ios-bug-priority-tree` in Claude Code or `$ios-bug-priority-tree` in Codex. The `agents/openai.yaml` in the skill folder supplies Codex's display name and default prompt; Claude Code ignores it.

## Use

It is written as a companion to a "find N distinct issues" `/goal` loop, and it expects to be loaded at the start of the hunt, before the first wave, since track assignment and the severity ladder apply from the beginning. The tree itself stays in the orchestrator's context. Each finder gets only its own track's section, never the whole file.

## Requirements

Xcode command line tools (`xcrun simctl`), `ffmpeg` for the video and frame-stepping steps, `sips` for downscaling screenshots, and an MCP server that can drive the simulator. The quoted tool names are [ios-simulator-mcp](https://github.com/joshuayoes/ios-simulator-mcp)'s, and the timing procedure assumes `idb` is available; swap in your own server's equivalents if you use a different one.

On Codex, subagents need the `multi_agent` feature, which is stable and on by default (`codex features list | grep multi_agent`). On Claude Code, no extra setup is needed.

Every command in the file was checked against Xcode on macOS in August 2026; re-check if your Xcode is much newer or older.

## License

MIT
