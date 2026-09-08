# Project Plan — "A minimum version of Braintrust"

Use this file as input when running `/gsd:new-project` — it pre-answers most of the Discuss/Requirements questions so GSD doesn't need to ask from scratch.

## Goal

A portfolio project for repositioning toward AI Engineering. Not a product to sell — the goal is to **deeply understand the agent infrastructure layer** (tracing, eval, observability, recovery) by self-building a minimum version of Braintrust, then writing an architecture teardown as the main interview artifact.

It directly realizes the plan already laid out in [`ai-engineering-brain`](https://github.com/haivx/ai-engineering-brain/tree/main/concepts/agent-infra) (state/memory tiering, eval-as-infra, observability, checkpoint/recovery).

## Known technical constraints / preferences

- Language: **TypeScript** throughout (agent, tracing, eval, dashboard) — one runtime, one repo, no serialization overhead between Python and TS
- Familiar stack for the dashboard: Next.js/TypeScript, Supabase
- The test agent (Phase 0) is hand-written using the Claude API, **no framework** (no Vercel AI SDK, no Mastra) — the point is to understand the tool-calling loop at a primitive level first
- Referenced "Harness Engineering & Agent Orchestration" (Scott Moss / Hendrixer, Frontend Masters) as a comparison point — not copying the course's architecture directly
- Suggested test agent: a small code-review agent (tools: read_file, run_linter, run_tests, search_codebase) — has real tool-calling and clear ground truth for eval (lint/test pass-fail)

## Roadmap (phase-level, for GSD to generate the detailed per-phase PLAN.md)

**Phase 0 — Test agent**
Hand-build a basic agent loop (model decides on a tool → calls it → gets the result → repeats) to serve as the subject for tracing/eval in later phases. No logging/framework yet. Deliverable: an agent runnable from the CLI.

**Phase 1 — Tracing**
Define a trace schema (spans, cost, latency, parent-child relationships). Wrap the Phase 0 agent so every step auto-logs to a trace store (Supabase/SQLite). Deliverable: one complete, queryable trace JSON per run.

**Phase 2 — Eval harness**
A small dataset (10–20 cases) plus a scorer combining hard assertions and LLM-as-judge. Run evals across agent/prompt versions and compare scores. Deliverable: an `eval run` command producing a comparison table.

**Phase 3 — Checkpoint/recovery + quality gate**
Persist state between steps so a failed run can resume instead of restarting. A gate that warns when eval score drops below a threshold. This is the hardest phase — the core focus is "determinism at the edge of a non-deterministic system."

**Phase 4 — Dashboard + demo + teardown**
A Next.js UI showing traces and eval scores over time. Prepare a concrete demo case (a bug → the trace explaining why → the eval catching the regression). Write the architecture teardown — the main artifact for interviews.

## Out of scope

- No need for a high-performance trace store (not Brainstore-level)
- No multi-tenancy, no SOC2/compliance
- No need to support multiple agent frameworks — just the one hand-built agent as the subject