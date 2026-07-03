# claude-skills-kit

A [Claude Code](https://docs.claude.com/en/docs/claude-code) plugin marketplace of custom skills. Each plugin packages a `SKILL.md` that Claude Code loads automatically when its description matches what you're asking for, so you can invoke it by name (`/skill-name`) or just by describing the task in natural language.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) CLI, desktop app, VS Code/JetBrains extension, or claude.ai/code
- Claude Code's plugin system enabled (default in recent versions)

This marketplace does **not** work with the general claude.ai web interface or the Claude mobile/desktop consumer apps — those use a separate, unrelated "Agent Skills" feature (Settings → Features → Skills) with its own upload mechanism.

## Install

```
/plugin marketplace add claude-skills-kit/claude-skills-kit
/plugin install fable-advisor@claude-skills-kit
```

Update later with:

```
/plugin marketplace update
```

## Plugins

### fable-advisor

Consult Fable 5 — the strongest, most expensive model available in a Claude Code session — as a one-shot planning advisor, without letting it become a quota drain.

**Why it exists:** on a Max plan, Fable's quota is a large share of the total budget. Calling it for routine work wastes that budget; never calling it means losing a second opinion on the hard, high-stakes, or hard-to-reverse decisions where it earns its cost. This skill encodes a repeatable process so that judgment call doesn't have to be re-made from scratch every time.

**How it works** — a five-stage flow:

1. **Gate** — decide whether the task actually warrants Fable. Defaults to skipping Fable for scoped bug fixes, cosmetic changes, or anything answerable from existing code/docs. Escalates to Fable for cross-cutting changes, irreversible/high-blast-radius work (auth, migrations, concurrency), genuinely novel problems, or when the user asks for Fable directly.
2. **Brief** — write a tight, curated brief (goal, constraints, absolute paths, prior attempts, acceptance criteria) instead of dumping the whole conversation — Fable starts cold on purpose, and this keeps the call cheap.
3. **Spawn** — run Fable as a read-only planning subagent (no file-edit access), so it can only produce a plan, never silently make changes.
4. **Implement** — the main model executes Fable's plan, checking it first against anything the user explicitly asked for (the user's instructions always win over the plan).
5. **Resume** — if implementation gets stuck, resume the same Fable session rather than re-briefing from scratch, with a built-in stop so a stuck task doesn't spiral into repeated paid calls.

**Usage:** invoke with `/fable-advisor`, or just say "ask fable" / "ถาม fable" / "consult fable" when you want a second opinion on an architecture choice, a hard bug, or a high-stakes change.

## Repository structure

```
claude-skills-kit/
├── .claude-plugin/
│   └── marketplace.json        # marketplace manifest listing all plugins below
└── fable-advisor/
    ├── .claude-plugin/
    │   └── plugin.json         # plugin manifest (name, version, author)
    └── skills/
        └── fable-advisor/
            └── SKILL.md         # the actual skill definition Claude Code loads
```

## Adding a new plugin to this marketplace

1. Create `<plugin-name>/.claude-plugin/plugin.json` and `<plugin-name>/skills/<skill-name>/SKILL.md`.
2. Add an entry to `.claude-plugin/marketplace.json` pointing `source` at `./<plugin-name>`.
3. Commit and push — anyone who already added this marketplace picks it up via `/plugin marketplace update`.

## Contributing

Issues and pull requests are welcome. Keep each plugin self-contained under its own top-level directory so the marketplace manifest stays a simple index.
