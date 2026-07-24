---
name: agent-advisor
description: |
  Consult a cold-context advisor subagent — Fable 5 or a fresh Opus — for a
  plan, a diagnosis, a critique, or a decision, while keeping Fable quota
  usage low. Use when the user says "ask fable", "ถาม fable", "ask opus",
  "ถาม opus", "consult an advisor", "ปรึกษา advisor", "agent-advisor", or asks
  for a second opinion / architecture plan / debugging strategy / critique of
  an existing plan on a genuinely hard, high-stakes, or hard-to-reverse
  problem. Runs a gate first to decide whether the task needs an advisor at
  all (an explicit user request always passes the gate), picks the tier and
  the consult mode, hands it a curated brief (not the raw transcript), checks
  the answer against the user's explicit instructions, and routes
  implementation back to the current main model. Do NOT invoke for small,
  clearly-scoped, low-risk work — the gate exists precisely to keep those off
  the advisors.
compatibility: claude-code
version: 2.0.0
allowed-tools:
  - Agent
  - SendMessage
  - AskUserQuestion
  - Read
  - Grep
---

# Agent Advisor: cold-context second opinions, spent economically

A fresh `Agent` subagent sees only what you hand it. That is the whole
mechanism this skill sells: an advisor with no memory of this conversation,
no attachment to the approach already tried, and — at the top tier — more raw
capability than the main model.

Two advisor tiers:

- **Fable 5** — when the task needs *more capability than the main model has*.
  Fable is the strongest and most expensive model available; on a Max 20x plan
  its quota is roughly half the total budget, so it gets spent like a scarce
  resource: for genuine planning/debugging leverage, never for routine work.
- **A fresh Opus subagent** — when the task needs *a view the main model can't
  have*. The main model is often already Opus; when it is, an Opus subagent is
  **not** a capability upgrade and **not** a cheaper tier. Its only value is
  the cold context — no anchoring on this transcript, no sunk cost in an
  approach already tried — or standing in when Fable is unavailable. If
  neither applies, spawn nothing and think it through yourself.

**Not a replacement for the built-in `advisor()` tool** — a different
instrument. `advisor()` sees the whole transcript automatically, costs no
briefing, and offers no model choice; reach for it as a continuous
checkpoint ("am I on track, did I miss something"). This skill costs a brief
and lets you choose both what the advisor sees and which model answers; reach
for it when the transcript *is* the problem (anchoring, sunk cost in an
approach already tried) or when you need above-main-model capability. The
one-line rule: value from someone seeing **everything you did** → `advisor()`.
Value from someone seeing **only the essentials, fresh** → this skill.

**Global rules**
- **Budget:** at most two advisor calls per task (one consult + one unblock)
  without telling the user first. After two passes, decide with the user
  rather than ping-ponging with a subagent.
- **Decision precedence:** user instructions > this skill's rules > the
  advisor's answer. The advisor's output is advice, not authority — the user
  always wins.
- **Advisors advise by default**, they don't implement — the main model
  implements.

Phases: **Gate → Mode → Brief → Spawn → Use**, plus a **Resume** path for
follow-ups.

## Phase 1 — Gate

**Consult an advisor if ANY are true:**
- User explicitly requested one (Fable or Opus)
- Architecture or long-term design decision with real tradeoffs
- Change is cross-cutting, or its blast radius is large and hard to reverse
- Small change, but a mistake would be costly, hard to detect, or
  irreversible (auth/crypto/payments, destructive data migrations,
  concurrency, production infrastructure)
- Problem is genuinely novel to this codebase — no existing pattern to copy
- Two genuinely different attempts have failed and the root cause is still
  unclear
- Requirements are technically ambiguous enough that a plan should exist
  before any code is touched

**Otherwise: skip.** Do not consult an advisor for routine implementation,
obvious bug fixes with a small blast radius, cosmetic changes, replicating an
existing pattern in the codebase, or anything answerable from docs/code/
project memory.

**If the gate passes, pick the tier — in this order:**

1. **User named a model** ("ask fable" / "ถาม opus") → use that one, no
   deliberation.
2. Does the task need **more capability than the main model has** —
   architecture with real tradeoffs, a genuinely novel problem, an
   irreversible blast radius? → **Fable**.
3. Does it need **a view the main model can't have** — an independent read of
   a plan this transcript is already anchored on, a cold diagnosis after two
   failed attempts, a review free of sunk cost? → **Opus subagent**.
4. Neither is clearly the fit → pick the closer of 2 and 3: a *design or plan*
   question goes to Fable, a *review or second-opinion* question to Opus. The
   Gate already decided **whether** to consult; this step only decides
   **which** — it can't send a passed task back to skip.

**Notes on applying the checklist:**
- **Intent before design:** if the ambiguity is about *what the user wants*
  (not how to build it), resolve it with `AskUserQuestion` first — a subagent
  starts cold and cannot ask the user. Then gate the clarified task.
- **Explicit request overrides everything above:** consult immediately, don't
  ask permission to skip. If the task looks routine, say so in one sentence
  *while proceeding* ("This looks routine, but consulting Fable as you
  asked"). Never silently substitute your own answer for a requested consult.
- **Already have an answer for this task from earlier in the session?** Don't
  re-gate or re-brief — go straight to **Resume** below.
- **Genuine toss-up on a high-stakes task:** ask the user once via
  `AskUserQuestion` rather than guessing.
- Otherwise, decide and state the reason in one sentence (e.g. "Scoped bug
  fix with an obvious cause — skipping the advisor and fixing it directly.").

**If SKIP:** proceed with the task yourself and stop here — do not run the
later phases.

## Phase 2 — Mode

A consult is only as good as the deliverable you name. Pick one mode and ask
for its shape explicitly in the brief:

| Mode | Reach for it when | Ask for |
|---|---|---|
| **plan** | design or strategy is the thing that's missing | ordered steps with file-level detail, plus explicitly-flagged open questions |
| **diagnose** | a bug survived two genuinely different attempts | root cause, the mechanism, a fix strategy, and what evidence would falsify the diagnosis |
| **critique** | a plan, diff, or spec already exists and you want it attacked | ranked concrete failure modes, each with the scenario that triggers it — "no material problems" must be an allowed answer |
| **decide** | two or more viable options with real tradeoffs | one pick, the deciding constraint named, and what fact would flip it |

**Anti-anchoring — mandatory for `critique` and `decide`.** Handing an advisor
your own plan while signalling that you like it buys agreement, not review.
So:
- Don't say which option you wrote, prefer, or already started building.
- Label options `A` / `B`, never "my approach" vs "the alternative".
- Omit your confidence level and any endorsement language.
- State that rejecting every option, or answering "no material problems", is a
  valid outcome.

If you catch yourself writing a brief that only makes sense if the advisor
agrees with you, you're in `plan` mode wearing a `critique` label — relabel it
or rewrite the brief.

## Phase 3 — Brief

**Prepare:**
- Verify every path you'll reference in the brief actually exists.
- Gather only the context the advisor can't get itself — it can Read/Grep once
  spawned, so don't pre-digest more than necessary.
- Batch related open questions into one brief. One consult covering three
  questions is far cheaper than three consults.

**Write the brief:** a fresh `Agent` call starts cold with zero memory of this
conversation — that's a feature here, not a limitation, and for an Opus
consult it is the *entire* point. The advisor only ever sees what you
deliberately hand it. Do not paste the whole conversation. The template is the
same for both tiers and all four modes.

```
## Brief for the advisor
**Goal:** <one or two sentences — what decision, plan, or diagnosis is needed>
**Repo / working dir:** <absolute path to the repo root — subagent cwds
  reset, so every path in this brief must be absolute>
**Constraints:** <stack, must-not-break, deadline, style/architecture rules;
  QUOTE the user's explicit instructions verbatim rather than paraphrasing>
**Context already known:** <relevant absolute file paths, prior decisions,
  relevant project memory — reference paths/names, don't paste full file
  contents unless essential>
**Specific question:** <the exact deliverable, in the shape Phase 2 names for
  the chosen mode. Add: "State your assumptions explicitly instead of asking
  questions back; list any decision that genuinely needs user input as an
  explicit open question.">

Include only if relevant:
**What's been tried / ruled out:** <why it failed. For `diagnose`: paste the
  exact error/stack trace/failing command output VERBATIM — the advisor
  cannot reproduce your interactive state>
**Acceptance:** <how success will be verified — the failing test to make
  pass, the command to run, the metric to hit>
**Out of scope:** <what NOT to touch or reconsider>
```

**Before spawning, check the brief in three directions:**
- **Enough:** could a cold advisor produce a useful answer from this brief
  alone? Any load-bearing fact that lives only in this conversation (a user
  decision, an error you saw, a constraint stated verbally) must be in the
  brief — the advisor cannot recover it.
- **Not too much:** include only information that changes the advisor's
  decision — don't summarize what it can discover itself. As a rough ceiling,
  most briefs run under ~300-400 words, **excluding** verbatim error output or
  schema snippets that are genuinely load-bearing — those earn their length;
  narrative padding does not.
- **Not leading** (`critique` / `decide` only): reread it once against the
  anti-anchoring rules in Phase 2.

## Phase 4 — Spawn

Tell the user in 1-2 sentences **which advisor and which mode** you're using
and why — do this **before** spawning, since the foreground call blocks until
it returns.

Default spawn (advisory-only, per Global rules):

```
Agent({
  description: "<3-5 word task description>",
  subagent_type: "Plan",       // no Edit/Write access — keeps the advisor advisory-only
  model: "fable",              // or "opus", per the tier picked in Phase 1
  run_in_background: false,     // default: you need the answer before continuing
  prompt: "<the brief from Phase 3>"
})
```

The only difference between the tiers is the `model` value. Use
`subagent_type: "general-purpose"` only if the user explicitly wants the
advisor to write code itself — otherwise `"Plan"` (can Read/Grep/explore,
cannot Edit/Write) prevents accidental drift into the advisor doing the
implementation work.

Set `run_in_background: true` only when you have genuinely independent prep to
do while the advisor thinks (e.g. reproducing the bug, running the test suite
to gather evidence for Phase 5) — otherwise stay foreground.

**If the spawn fails** (quota exhausted, model unavailable, tool error): tell
the user immediately and offer the choice — retry later, work with the main
model now, or, when Fable is the one that failed, fall back to an Opus
subagent for a cold read. Offering a different tier as an alternative is fine;
silently substituting one for a requested Fable consult is not.

Note the agent id/name the harness returns — you need it for Resume.

## Phase 5 — Use the answer

Before acting on it, run this checklist:

1. **It answers the question, in the mode you asked for:** a `critique` that
   comes back as a rewritten plan, or a `decide` that comes back as "it
   depends", missed the deliverable. If so, follow up via Resume step 1 scoped
   to the gap — don't act on a mismatched answer.
2. **User instructions win** (decision precedence, Global rules). If the
   answer contradicts something the user explicitly asked for, surface the
   conflict and the advisor's rationale and let the user decide — never
   silently follow the advisor over the user.
3. **Open questions:** if the answer flags decisions needing user input,
   resolve them with the user before touching code. Don't guess on
   load-bearing ones.
4. **Relay it:** give the user a 3-5 line summary of the approach and key
   decisions — always, not just for big answers. If it's high-stakes
   (destructive, hard to reverse, or large-scale), wait for the user's
   go-ahead before making edits; otherwise proceed.
5. **Make the verdict findable** when the consult actually changed the design:
   put the deciding reason in the commit message, or in project memory if it
   will outlive the commit. An unrecorded verdict gets re-litigated — and
   re-consulted — next session.

Then execute yourself (or delegate to a `sonnet`/`opus` subagent for large
mechanical portions — that's plain delegation, not an advisor call, and
doesn't count against the budget). Small deviations discovered during
implementation are normal — note them and continue. Do not re-consult an
advisor to double-check routine implementation steps — that defeats the point
of the gate.

## Resume — when implementation gets stuck

**Resume priority:**

1. **`SendMessage` to the existing agent** (using the agent id from Phase 4).
   This works even after the advisor already returned its answer — a completed
   foreground agent auto-resumes on `SendMessage`. Scope the message to just
   the blocker: what you tried, the exact error/result verbatim, and the one
   question you need answered.
2. **New `Agent` call at the same tier** (`model: "fable"` or `"opus"`) — only
   if the original agent is genuinely gone or unrelated to the blocker. Write
   a fresh, narrow brief (Phase 3 style, scoped to the blocker) and quote the
   relevant excerpt of the original answer — a new advisor has never seen it
   and may otherwise re-derive or contradict it. Escalating Opus → Fable here
   is allowed when the blocker turns out to be the kind of problem step 2 of
   the tier check describes; say so to the user when you do.
3. **Ask the user before exceeding the two-call budget** (Global rules
   above) — at that point it's quota burning, not leverage. A blocker isn't
   the same as a broken plan: if the plan turns out fundamentally infeasible,
   summarize the state to the user and decide together instead of
   ping-ponging with a subagent.

Then go back to implementing with the main model.

## Quick reference

| Step | Tool | Model |
|---|---|---|
| Skip | — | main model only |
| Explicit request | — | always consult, at the model named |
| Gate check | (reasoning only; `AskUserQuestion` for intent) | main model |
| Tier check | (reasoning only; capability → Fable, cold view → Opus) | main model |
| Mode check | (reasoning only; plan / diagnose / critique / decide) | main model |
| Brief | (reasoning only; verify paths, check for leading language) | main model |
| Consult | `Agent` (`subagent_type: "Plan"`) | `fable` or `opus` |
| Answer check + summary to user | (reasoning only; user wins conflicts) | main model |
| Implement | direct edits / `Agent` | `sonnet` or `opus` |
| Unblock (max 1 without asking) | `SendMessage` to existing agent, or new narrow `Agent` | same tier (Opus → Fable escalation allowed) |

## Changelog

Entry format (binding on future entries): max 4 lines — `Added/Changed:`
summaries, plus `Rejected: <idea> — <one-clause reason>` for any declined
review suggestion so it isn't re-proposed. Long-form rationale goes in git
commit messages, never here.

- **2.0.0** — Renamed `fable-advisor` → `agent-advisor` (user request; two
  tiers now, not one). Added: four consult modes (plan/diagnose/critique/
  decide) each naming its deliverable, mandatory anti-anchoring rules for
  critique/decide, "make the verdict findable" as Phase 5 item 5, and a
  routing rule vs. the built-in `advisor()` tool.
- **1.4.0** — Added: second tier (fresh Opus subagent) with a two-question
  tier check, Opus as Fable-spawn fallback, "ask opus"/"ถาม opus" triggers;
  budget now counts two *advisor* calls total. Rejected: a separate Opus
  budget number — two caps drift apart.
- **1.3.1** — Changelog capped to the format above; entries retro-trimmed.
  Rejected: moving rationale to a separate DESIGN.md — rejection markers
  only inoculate if they're in the file reviewers actually see, and the
  ~300-token runtime cost doesn't clear the one-file default; long-form
  rationale belongs in git instead. No runtime change.
- **1.3.0** — Added: Brief both-directions completeness check; "the answer
  addresses the brief's question" as a first check item; offset by trimming
  duplicated advisory-only/precedence prose. Rejected: numeric anchors →
  quality language — anchors are the drift detectors; per-phase exit
  criteria — every proposed exit already exists in operative form.
- **1.2.0** — Editorial/scannability pass only: checklist-first Gate,
  Prepare/Write Brief split, priority-list Resume, global rules promoted,
  Quick Reference covers Skip and explicit-request paths. No behavioral
  change from 1.1.0.
- **1.1.0** — Single CONSULT-checklist Gate; spawn-failure fallback;
  two-call budget; always-relay summary; path verification. Rejected:
  demoting `SendMessage` in Resume — completed foreground agents remain
  addressable.
- **1.0.0** — Initial: Gate → Brief → Spawn → Implement, plus Resume.
