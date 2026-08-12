# ios-bug-priority-tree

An agent skill for [Claude Code](https://claude.com/claude-code) and [Codex](https://developers.openai.com/codex/cli) that runs a long iOS bug hunt with subagents against the iOS Simulator, and keeps finding real bugs after the easy ones are gone.

Point a few agents at an app and tell them to "find bugs" and you get one wave of easy findings, then diminishing returns, then a lot of tokens spent re-walking the same happy paths. This skill gives the orchestrator a plan for what to do next, and a shared definition of what counts as a bug so agents stop arguing with themselves about whether something is "just cosmetic."

Severity runs S1 to S4: S1 is a crash or data loss, S2 is a broken flow or wrong data, S3 is misleading state (the user cannot tell what happened), S4 is pure polish. S1 through S3 count toward the goal; S4 is capped at 2 so finders don't farm alignment nits. Most of the rest came out of watching runs go wrong: a verifier that spent 326 turns on a 5-step repro, agents dismissing a return-to-springboard as "a transition," phantom hangs caused by mistiming a tap rather than by the app.

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

Invoke it directly with `/ios-bug-priority-tree` in Claude Code or `$ios-bug-priority-tree` in Codex. It is written as a companion to a "find N distinct issues" `/goal` loop, and it expects to be loaded at the start of the hunt, before the first wave, since track assignment and the severity ladder apply from the beginning. The tree stays in the orchestrator's context; each finder gets only its own track's section, never the whole file.

## What it does

1. **Split** the hunt into four finder tracks that all run from wave one. Track A drives the app through the accessibility tree, hunting broken flows, lost input, and bad error states. Track B screenshots every screen and reads the pixels against a rubric (tap targets, contrast, truncation, keyboard avoidance, empty states), in light and dark mode. Track C reads the source with no simulator attached, so it runs in parallel for free. Track D measures: hangs over 1s, cold launch over 2s, memory that grows and never plateaus.
2. **Escalate** per track when a wave comes up empty, instead of repeating it. Track A moves to permission denial, deep links, push routing, hostile timezones, then RTL, then the expensive manual work. Track B moves to Dynamic Type extremes, pseudo-localization, increased contrast, fake status bars, then frame-stepping recorded animations to catch bugs that only exist mid-transition.
3. **Gate** Track C's findings behind a user-visible repro; a code smell is not a bug until someone reproduces it.
4. **Budget** the turns. Each agent turn re-reads the whole conversation, so a 300-turn agent costs far more than ten 30-turn agents. The skill caps finder turns, forbids screenshot re-reads, requires a fresh agent per wave, and holds reports to 25 lines.
5. **Verify** blind. Verifiers are barred from reading the source, the diff, or git history, because a verifier that finds the fix will rubber-stamp it. They drive the installed build and describe what they actually saw.

See [SKILL.md](skills/ios-bug-priority-tree/SKILL.md) for the full tree.

## Requirements

Xcode command line tools (`xcrun simctl`), `ffmpeg` for the video and frame-stepping steps, `sips` for downscaling screenshots, and an MCP server that can drive the simulator. The quoted tool names are [ios-simulator-mcp](https://github.com/joshuayoes/ios-simulator-mcp)'s, and the timing procedure assumes `idb` is available; swap in your own server's equivalents if you use a different one.

On Codex, subagents need the `multi_agent` feature, which is stable and on by default (`codex features list | grep multi_agent`). On Claude Code, no extra setup is needed. The `agents/openai.yaml` in the skill folder supplies Codex's display name and default prompt; Claude Code ignores it.

Every command in the file was checked against Xcode on macOS in August 2026; re-check if your Xcode is much newer or older.

## License

[MIT](LICENSE)
