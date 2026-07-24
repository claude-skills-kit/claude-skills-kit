# claude-skills-kit

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace of custom skills. Each plugin packages a `SKILL.md` that Claude Code loads automatically when its description matches what you're asking for, so you can invoke it by name (`/skill-name`) or just by describing the task in natural language.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) CLI, desktop app, VS Code/JetBrains extension, or claude.ai/code
- Claude Code's plugin system enabled (default in recent versions)

This marketplace does **not** work with the general claude.ai web interface or the Claude mobile/desktop consumer apps — those use a separate, unrelated "Agent Skills" feature (Settings → Features → Skills) with its own upload mechanism.

## Install

```
/plugin marketplace add claude-skills-kit/claude-skills-kit
/plugin install agent-advisor@claude-skills-kit
```

Update later with:

```
/plugin marketplace update
```

> **Renamed in v2.0.0:** this plugin was previously published as `fable-advisor`. If you installed it under the old name, run `/plugin uninstall fable-advisor` and install `agent-advisor` instead — the old plugin id no longer exists in the marketplace.

## Plugins

### agent-advisor

Consult a **cold-context advisor subagent** — Fable 5, or a fresh Opus — for a plan, a diagnosis, a critique, or a decision, without letting it become a quota drain.

**Why it exists:** a fresh subagent sees only what you hand it. That buys two different things. From Fable — the strongest, most expensive model in a Claude Code session — it buys capability, but on a Max plan Fable's quota is a large share of the total budget, so calling it for routine work is waste. From a fresh Opus it buys a *cold context*: no memory of the conversation, no attachment to the approach already tried. This skill encodes when each is worth it, so that judgment call doesn't get re-made from scratch every time.

**Two tiers:**

- **Fable 5** — when the task needs *more capability than the main model has*: architecture with real tradeoffs, a genuinely novel problem, an irreversible blast radius.
- **A fresh Opus subagent** — when the task needs *a view the main model can't have*: an independent read of a plan the transcript is already anchored on, a cold diagnosis after two failed attempts. Note that when the main model is already Opus, this is neither a capability upgrade nor a cheaper tier — only the cold context justifies it.

**How it works** — a five-phase flow, plus a Resume path:

1. **Gate** — decide whether the task warrants an advisor at all. Defaults to skipping for scoped bug fixes, cosmetic changes, or anything answerable from existing code/docs. Escalates for cross-cutting changes, irreversible/high-blast-radius work (auth, migrations, concurrency), genuinely novel problems, or when the user asks directly. A four-step tier check then picks Fable or Opus.
2. **Mode** — name the deliverable: `plan` (ordered steps with file-level detail), `diagnose` (root cause, mechanism, what would falsify it), `critique` (ranked failure modes with triggering scenarios), or `decide` (one pick, the deciding constraint, what would flip it). `critique` and `decide` carry mandatory anti-anchoring rules — never reveal which option you wrote or prefer, label options A/B, and leave "no material problems" available as an answer. Handing an advisor your own plan while signalling you like it buys agreement, not review.
3. **Brief** — write a tight, curated brief (goal, constraints, absolute paths, prior attempts, acceptance criteria) instead of dumping the whole conversation. The advisor starts cold on purpose, and this keeps the call cheap.
4. **Spawn** — run the advisor as a read-only planning subagent (no file-edit access), so it can only produce an answer, never silently make changes.
5. **Use** — the main model acts on the answer, checking it first against anything the user explicitly asked for (the user's instructions always win), and records a design-changing verdict where it will be found again.

**Resume** — if implementation gets stuck, resume the same advisor session rather than re-briefing from scratch, with a built-in stop so a stuck task doesn't spiral into repeated paid calls.

**Usage:** invoke with `/agent-advisor`, or just say "ask fable" / "ถาม fable" / "ask opus" / "consult an advisor" when you want a second opinion on an architecture choice, a hard bug, or a high-stakes change.

## Repository structure

```
claude-skills-kit/
├── .claude-plugin/
│   └── marketplace.json        # marketplace manifest listing all plugins below
└── agent-advisor/
    ├── .claude-plugin/
    │   └── plugin.json         # plugin manifest (name, version, author)
    └── skills/
        └── agent-advisor/
            └── SKILL.md         # the actual skill definition Claude Code loads
```

## Adding a new plugin to this marketplace

1. Create `<plugin-name>/.claude-plugin/plugin.json` and `<plugin-name>/skills/<skill-name>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json` pointing `source` at `./<plugin-name>`.
3. Commit and push — anyone who already added this marketplace picks it up via `/plugin marketplace update`.

## Contributing

Issues and pull requests are welcome. Keep each plugin self-contained under its own top-level directory so the marketplace manifest stays a simple index.
