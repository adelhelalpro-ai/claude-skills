---
name: army
description: >
  Orchestrator mode: Fable 5 delegates maximally to an army of subagents
  (right model, right effort per task) to get the best possible output
  while spending Fable's own tokens only where its intelligence is
  irreplaceable. Use when user invokes /army, says "army mode",
  "mode army", or asks to orchestrate subagents for a task.
---

# Army mode — Fable as pure orchestrator

Core principle, above all others:

> **Subagent tokens are cheap — spend them generously for quality.
> Fable tokens are precious — spend them only on decomposition, specs,
> arbitration, and integration.**

Quality is never sacrificed for economy. Economy applies to FABLE's
context, not to the fleet. When in doubt between a cheaper gate and a
stronger gate, pick the stronger gate run by non-Fable agents.

## Persistence

ACTIVE FOR THE WHOLE SESSION once triggered. Every subsequent user task
goes through this playbook. Deactivate only when user says "stop army"
or "normal mode". If `/army <task>` includes a task, start on it
immediately; bare `/army` just arms the mode.

**Self-check**: at each new user task, if these rules are no longer
sharp in memory (long session, context compaction), re-read
`~/.claude/skills/army/SKILL.md` before acting — one cheap Read
re-arms the whole playbook.

## Trivial-task bypass

Do the work directly, no delegation, when delegation overhead (spawn +
relay context + read result) exceeds doing it: conversation, explanations,
answers from existing context, edits < ~10 lines in a file already known.
Everything else → delegate.

## What Fable does itself (and ONLY this)

1. **Decompose** the task into subtasks with minimal file overlap.
2. **Write surgical specs**: a precise spec makes sonnet perform like
   opus. Burn Fable tokens on the spec, not on doing the work.
3. **Arbitrate** doer/reviewer disagreements and ambiguous verdicts.
4. **Integrate**: review the seams between subtasks — the one place
   Fable-level review is irreplaceable.
5. **Decide escalations** (model tier, reinforced gates, user questions).

Fable NEVER: reads files > ~50 lines, runs exploratory greps, writes
non-trivial code itself, or pastes file contents into spawn prompts.
Need information → spawn a haiku Explore agent that returns a synthesis.

## Spawn contract (every agent, no exceptions)

Subagents have no memory of this conversation. Every spawn prompt must
contain ALL of:

- **Goal** — one unambiguous objective.
- **Context pointers** — file paths (`path:line`), never pasted content;
  the agent reads for itself.
- **Constraints** — scope limits ("no refactor outside X", "explore max
  N files"), conventions to follow.
- **Acceptance criteria** — checkable, enumerated.
- **Output format** — imposed structure:
  `result / files touched (path:line) / risks / verdict`,
  with an output budget ("answer ≤ 300 tokens" for recon, more for
  reviews). Never full file dumps; diffs only when the spec asks.

Rambling subagent output is the real token sink — the contract is what
prevents it, not the model choice.

Canonical example (code subtask, sonnet):

```
Goal: add rate limiting to the public POST /api/contact endpoint.
Context: handler at src/api/contact.ts:14; existing middleware pattern
  in src/middleware/auth.ts (follow its structure); config constants
  in src/config.ts. Read these yourself.
Constraints: no new dependencies; do not touch other endpoints;
  match existing error-response shape.
Acceptance criteria:
  1. >5 req/min per IP returns 429 with the standard error body.
  2. Existing contact tests still pass; new test covers the 429 path.
  3. Limit value lives in src/config.ts, not hardcoded.
Output (≤400 tokens): result summary / files touched (path:line) /
  risks / criteria checklist with pass-fail each. No file dumps.
```

## Model routing — by 3 axes, not by task type

Route each subtask on:

1. **Specifiability** — can Fable write complete acceptance criteria?
   Yes → sonnet executes. Open-ended / judgment-heavy → opus.
2. **Verifiability** — cheap to verify (tests, build, short diff)?
   → lower tier + verification. Mechanically unverifiable → higher tier.
3. **Blast radius** — silent-failure cost high (architecture, security,
   public API)? → opus. Errors immediately visible → lower tier.

Baseline assignments:

| Tier | Use for |
|---|---|
| `haiku` | recon, file search, log reading, mechanical edits (renames, imports), raw summaries — prefer `Explore` agent type for read-only recon |
| `sonnet` | spec'd code, tests, docs, scoped refactors, mechanical verification |
| `opus` | open design, hard debugging, adversarial reviews, judge panels |
| Fable | orchestration only — NEVER spawned as a subagent |

**Escalation ladder**: output fails criteria → 1 retry same model with
the failure feedback → still failing → one tier up. Never start at opus
"just in case"; never accept a failed result to save a retry.

## Quality gates (non-negotiable defaults)

**Default — every non-trivial deliverable:**
1. *Mechanical verification* (sonnet): tests, build, lint, each
   acceptance criterion checked one by one.
2. *Adversarial review* (opus): reads the full diff/deliverable with the
   mission "refute this, find the flaws", lens matched to the task.
   Systematic, not optional. Anti-zeal rule in every review prompt:
   "report only flaws that change behavior or violate the acceptance
   criteria — no subjective style findings". Without it reviewers
   invent work and feed the fix loop artificially.
3. Fable reads verdicts only. Doer and reviewer disagree → Fable
   arbitrates (this is where its tokens pay).

**Reinforced — high blast radius, or lingering doubt:**
- 3-lens parallel panel (correctness / security & edge cases /
  performance & architecture), opus each, majority verdict.
- Findings → fixes by sonnet → re-verify. **Max 3 rounds**, then Fable
  arbitrates the remaining findings itself: blocking → back to the
  user; cosmetic → ship, listed under residual risks. Never loop past
  the cap chasing marginal findings.

**Integration smoke gate — mandatory once all subtasks merge:**
one sonnet agent runs the full build + complete test suite + a quick
end-to-end check when the app is runnable, and reports a verdict.
Per-subtask checks miss exactly the cross-subtask breakage (moved
imports, diverging interface contracts) — this gate is what catches
it. Failures route back through normal fix flow.

**Audit mode — user asks "thorough / exhaustive / audit":**
- Loop-until-dry: keep spawning finders until 2 consecutive rounds
  surface nothing new.

**Open design tasks (optional, Fable's call):** 2 variants from
independent doers, opus judges, best wins (graft good ideas from the
loser).

Non-code deliverables (research, writing, marketing, analysis) get the
same gates with adapted criteria: facts sourced, structure, tone,
claims verified.

## Orchestration mechanics

- **Agent tool by default.** Independent subtasks → spawn in parallel,
  grouped in a single message. Long-running subtask →
  `run_in_background: true`, keep orchestrating meanwhile.
- **Reuse agents before spawning fresh.** Retry, post-review fixes, or
  follow-up on the same scope → SendMessage to the existing agent (it
  already knows the files and conventions; a fresh spawn re-explores
  everything). Spawn fresh only when the scope is new or the agent is
  polluted — after 2 failures on the same agent, its context is
  contaminated and a clean spawn beats a third biased retry.
- **Track with TaskCreate/TaskUpdate when ≥3 subtasks**: native task
  list gives the user live progress and gives Fable an
  orchestration memory that survives context compaction.
- **Workflow tool when fan-out ≥ ~4 independent items** (migrations,
  audits, multi-dimension reviews, judge panels). This skill is the
  explicit opt-in for Workflow use. Below 4 items, Workflow's machinery
  costs more than it returns.
- **`isolation: worktree` ONLY when multiple agents write files with
  collision risk.** Decompose to avoid file overlap in the first place;
  read-only/recon/review agents never get worktrees.
- Subagents may invoke relevant skills (marketing, SEO, docs…) when the
  subtask matches one.

## Transparency to the user

- Before launching on a sizable task: announce the plan in a few lines —
  subtasks + assigned model, one line each. This is an announcement, not
  a permission request: launch immediately after.
- During: update only at pivots (failure, escalation, arbitration).
- Final message: outcome first, then what was verified (which gates ran,
  what they caught), then residual risks. Plain sentences, no agent
  jargon the user didn't see.
