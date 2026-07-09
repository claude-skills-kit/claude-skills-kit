---
name: fable-advisor
description: |
  Consult Fable 5 as a one-shot planning advisor while keeping quota usage low.
  Use when the user says "ask fable", "ถาม fable", "consult fable", "fable-advisor",
  or asks for a second opinion / architecture plan / debugging strategy on a
  genuinely hard, high-stakes, or hard-to-reverse problem. Runs a gate first to
  decide whether the task actually needs Fable (an explicit user request to
  consult Fable always passes the gate), then hands Fable a curated brief (not
  the raw transcript), gets back a plan, checks it against the user's explicit
  instructions, and routes implementation to the current main model
  (Opus/Sonnet). Do NOT invoke for small, clearly-scoped, low-risk work — the
  gate exists precisely to keep those off Fable.
compatibility: claude-code
version: 1.3.1
allowed-tools:
  - Agent
  - SendMessage
  - AskUserQuestion
  - Read
  - Grep
---

# Fable Advisor: economical use of Fable 5

Fable 5 is the strongest, most expensive model available to this session. On a
Max 20x plan its quota is roughly half of the total budget, so it must be
spent like a scarce resource: for genuine planning/debugging leverage, never
for routine work the main model already handles well.

**Global rules**
- **Budget:** at most two Fable calls per task (one consult + one unblock)
  without telling the user first.
- **Decision precedence:** user instructions > this skill's rules > Fable's
  plan. Fable's output is advice, not authority — the user always wins.
- **Fable is a planning advisor by default**, not an implementer — it plans,
  the main model implements.

This skill runs four phases: **Gate → Brief → Spawn → Implement**, plus a
**Resume** path for follow-ups.

## Phase 1 — Gate

**Consult Fable if ANY are true:**
- User explicitly requested Fable
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

**Otherwise: skip.** Do not consult Fable for routine implementation, obvious
bug fixes with a small blast radius, cosmetic changes, replicating an
existing pattern in the codebase, or anything answerable from docs/code/
project memory.

**Notes on applying the checklist:**
- **Intent before design:** if the ambiguity is about *what the user wants*
  (not how to build it), resolve it with `AskUserQuestion` first — Fable
  starts cold and cannot ask the user. Then gate the clarified task.
- **Explicit request overrides everything above:** consult immediately, don't
  ask permission to skip. If the task looks routine, say so in one sentence
  *while proceeding* ("This looks routine, but consulting Fable as you
  asked"). Never silently substitute your own answer for a requested consult.
- **Already have a plan for this task from earlier in the session?** Don't
  re-gate or re-brief — go straight to **Resume** below.
- **Genuine toss-up on a high-stakes task:** ask the user once via
  `AskUserQuestion` rather than guessing.
- Otherwise, decide and state the reason in one sentence (e.g. "Scoped bug
  fix with an obvious cause — skipping Fable and fixing it directly.").

**If SKIP:** proceed with the task yourself and stop here — do not run
Phases 2-4.

## Phase 2 — Brief

**Prepare:**
- Verify every path you'll reference in the brief actually exists.
- Gather only the context Fable can't get itself — it can Read/Grep once
  spawned, so don't pre-digest more than necessary.
- Batch related open questions into one brief. One consult covering three
  questions is far cheaper than three consults.

**Write the brief:** a fresh `Agent` call starts cold with zero memory of
this conversation — that's a feature here, not a limitation. Fable only ever
sees what you deliberately hand it. Do not paste the whole conversation.

```
## Brief for Fable
**Goal:** <one or two sentences — what decision or plan is needed>
**Repo / working dir:** <absolute path to the repo root — subagent cwds
  reset, so every path in this brief must be absolute>
**Constraints:** <stack, must-not-break, deadline, style/architecture rules;
  QUOTE the user's explicit instructions verbatim rather than paraphrasing>
**Context already known:** <relevant absolute file paths, prior decisions,
  relevant project memory — reference paths/names, don't paste full file
  contents unless essential>
**Specific question:** <the exact plan, diagnosis, or decision Fable should
  produce — be precise about the deliverable shape, e.g. "a step-by-step
  implementation plan with file-level detail" or "root-cause diagnosis plus
  a fix strategy". Add: "State your assumptions explicitly instead of asking
  questions back; list any decision that genuinely needs user input as an
  explicit open question in the plan.">

Include only if relevant:
**What's been tried / ruled out:** <why it failed. For debugging: paste the
  exact error/stack trace/failing command output VERBATIM — Fable cannot
  reproduce your interactive state>
**Acceptance:** <how success will be verified — the failing test to make
  pass, the command to run, the metric to hit>
**Out of scope:** <what NOT to touch or reconsider>
```

**Before spawning, check the brief in both directions:**
- **Enough:** could a cold Fable produce a useful plan from this brief alone?
  Any load-bearing fact that lives only in this conversation (a user
  decision, an error you saw, a constraint stated verbally) must be in the
  brief — Fable cannot recover it.
- **Not too much:** include only information that changes Fable's planning
  decision — don't summarize what Fable can discover itself. As a rough
  ceiling, most briefs run under ~300-400 words, **excluding** verbatim
  error output or schema snippets that are genuinely load-bearing — those
  earn their length; narrative padding does not.

## Phase 3 — Spawn

Tell the user in 1-2 sentences that you're consulting Fable and why — do this
**before** spawning, since the foreground call blocks until Fable returns.

Default spawn (advisory-only, per Global rules):

```
Agent({
  description: "<3-5 word task description>",
  subagent_type: "Plan",       // no Edit/Write access — keeps Fable advisory-only
  model: "fable",
  run_in_background: false,     // default: you need the plan before continuing
  prompt: "<the brief from Phase 2>"
})
```

Use `subagent_type: "general-purpose"` with `model: "fable"` only if the user
explicitly wants Fable to write code itself — otherwise `"Plan"` (can
Read/Grep/explore, cannot Edit/Write) prevents accidental drift into Fable
doing paid implementation work.

Set `run_in_background: true` only when you have genuinely independent prep
to do while Fable thinks (e.g. reproducing the bug, running the test suite to
gather evidence for Phase 4) — otherwise stay foreground.

**If the spawn fails** (quota exhausted, model unavailable, tool error): tell
the user immediately and offer the choice — plan with the main model now, or
retry Fable later. Never silently downgrade a requested Fable consult to a
main-model answer.

Note the agent id/name the harness returns — you need it for Resume.

## Phase 4 — Implement

Before executing, run this checklist on Fable's plan:

1. **It answers the question:** the plan addresses the brief's Specific
   question and has no obvious internal contradictions. If Fable answered a
   different question, follow up via Resume step 1 scoped to the gap — don't
   implement a mismatched plan.
2. **User instructions win** (decision precedence, Global rules). If the
   plan contradicts something the user explicitly asked for, surface the
   conflict and Fable's rationale and let the user decide — never silently
   follow Fable over the user.
3. **Open questions:** if the plan flags decisions needing user input,
   resolve them with the user before touching code. Don't guess on
   load-bearing ones.
4. **Relay the plan:** give the user a 3-5 line summary of the approach and
   key decisions — always, not just for big plans. If the plan is
   high-stakes (destructive, hard to reverse, or large-scale), wait for the
   user's go-ahead before making edits; otherwise proceed.

Then execute the plan yourself (or delegate to a `sonnet`/`opus` subagent for
large mechanical portions). Small deviations from the plan discovered during
implementation are normal — note them and continue. Do not re-consult Fable
to double-check routine implementation steps — that defeats the point of the
gate.

## Resume — when implementation gets stuck

**Resume priority:**

1. **`SendMessage` to the existing agent** (using the agent id from Phase 3).
   This works even after Fable already returned its plan — a completed
   foreground agent auto-resumes on `SendMessage`. Scope the message to just
   the blocker: what you tried, the exact error/result verbatim, and the one
   question you need answered.
2. **New `Agent` call with `model: "fable"`** — only if the original agent is
   genuinely gone or unrelated to the blocker. Write a fresh, narrow brief
   (Phase 2 style, scoped to the blocker) and quote the relevant excerpt of
   Fable's original plan — a new Fable has never seen it and may otherwise
   re-derive or contradict it.
3. **Ask the user before exceeding the two-call budget** (Global rules
   above) — at that point it's quota burning, not leverage. A blocker isn't
   the same as a broken plan: if the plan turns out fundamentally infeasible,
   summarize the state to the user and decide together instead of
   ping-ponging with Fable.

Then go back to implementing with the main model.

## Quick reference

| Step | Tool | Model |
|---|---|---|
| Skip | — | main model only |
| Explicit request | — | always consult |
| Gate check | (reasoning only; `AskUserQuestion` for intent) | main model |
| Brief | (reasoning only; verify paths) | main model |
| Consult | `Agent` (`subagent_type: "Plan"`) | `fable` |
| Plan check + summary to user | (reasoning only; user wins conflicts) | main model |
| Implement | direct edits / `Agent` | `sonnet` or `opus` |
| Unblock (max 1 without asking) | `SendMessage` to existing agent, or new narrow `Agent` | `fable` |

## Changelog

Entry format (binding on future entries): max 4 lines — `Added/Changed:`
summaries, plus `Rejected: <idea> — <one-clause reason>` for any declined
review suggestion so it isn't re-proposed. Long-form rationale goes in git
commit messages, never here.

- **1.3.1** — Changelog capped to the format above; entries retro-trimmed.
  Rejected: moving rationale to a separate DESIGN.md — rejection markers
  only inoculate if they're in the file reviewers actually see, and the
  ~300-token runtime cost doesn't clear the one-file default; long-form
  rationale belongs in git instead. No runtime change.
- **1.3.0** — Added: Brief both-directions completeness check; "plan
  answers the brief's question" as Phase 4 item 1; offset by trimming
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
