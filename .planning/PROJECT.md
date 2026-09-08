# Tracewell — A Minimum Version of Braintrust

## What This Is

A self-built, minimum version of Braintrust: an agent infrastructure layer covering tracing,
evaluation, observability, and checkpoint/recovery. The subject under observation is a small,
hand-written code-review agent built directly against the Claude API with no agent framework.
This is a portfolio and learning project for repositioning toward AI Engineering — not a product
to sell. Its audience is interviewers and the author.

## Core Value

A live, screen-shareable demo of the full loop — a bug appears, the trace explains why, the eval
catches the regression — backed by an architecture teardown the author can defend in depth.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] A hand-written agent loop (model → tool call → result → repeat) runnable from the CLI
- [ ] A trace schema covering spans, parent-child nesting, latency, and cost
- [ ] Automatic trace capture wrapping the agent, producing one queryable trace per run
- [ ] A fixture repo of seeded bugs providing deterministic eval ground truth
- [ ] An eval harness over 10-20 cases combining hard assertions with LLM-as-judge
- [ ] Comparison of two named runs, enough to demonstrate regression detection
- [ ] Checkpoint/resume so a failed run continues instead of restarting
- [ ] A quality gate that warns when eval score drops below threshold
- [ ] A Next.js dashboard with a trace waterfall viewer, run comparison diff, and cost/token breakdown
- [ ] A prepared demo case: bug → trace → eval catches regression
- [ ] The architecture teardown document

### Out of Scope

- High-performance trace store (Brainstore-class) — irrelevant at this scale; the learning is in the schema, not the storage engine
- Multi-tenancy, SOC2, compliance — no external users
- Support for multiple agent frameworks — one hand-built agent is the whole subject
- Agent frameworks for the agent itself (Vercel AI SDK, Mastra) — understanding the tool-calling loop at a primitive level is the point
- Eval-scores-over-time trend view — deferred to v2; a trend line needs many runs to say anything, and this project will produce few
- Full prompt/agent versioning model — comparison is built only as deep as the demo requires

## Context

- Directly realizes the plan laid out in [`ai-engineering-brain`](https://github.com/haivx/ai-engineering-brain/tree/main/concepts/agent-infra): state/memory tiering, eval-as-infra, observability, checkpoint/recovery.
- Source brief: `planning/PLAN.md` in this repo.
- "Harness Engineering & Agent Orchestration" (Scott Moss / Hendrixer, Frontend Masters) is a comparison point, not a template to copy.
- Greenfield repository — no existing code at initialization.
- Working rhythm: part-time, evenings and weekends, roughly 4-8 weeks. Phases are sized to finish in one or two sittings.

## Constraints

- **Tech stack**: TypeScript throughout — agent, tracing, eval, and dashboard — One runtime and one repo; avoids serialization overhead and context-switching between Python and TS
- **Tech stack**: Next.js + Supabase for the dashboard — Already familiar; the goal is to spend learning budget on the infra layer, not the UI layer
- **Tech stack**: No agent framework for the agent under test — The tool-calling loop is the thing being learned; a framework would hide it
- **Timeline**: Part-time over weeks — Phases must produce standalone value, since sessions are short and interruptible
- **Dependencies**: Claude API — The agent and the LLM-as-judge scorer both require it

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| SQLite first, migrate to Supabase at the dashboard phase | Local iteration without network for Phases 1-3; pay the one migration when the dashboard actually needs Postgres | — Pending |
| Fixture repo of seeded bugs as eval ground truth | Deterministic and checkable ("did it find bug #7?"); hand-labelling real diffs gives fuzzier signal | — Pending |
| Split checkpoint/recovery and the eval quality gate into separate phases | It is the hardest and most interesting work; finer slices give clearer wins and fit part-time sittings | — Pending |
| Version comparison built only as deep as the demo needs | A full versioning model is Braintrust's product surface, not the learning objective | — Pending |
| Dashboard shows traces, run diffs, and cost — not score trends | A trend line is meaningless across few runs | — Pending |
| Code-review agent as the subject | Real tool-calling (read_file, run_linter, run_tests, search_codebase) plus objective ground truth from lint/test pass-fail | — Pending |
| Git branching per phase | Clean per-phase diffs the teardown can cite directly | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-08 after initialization*
