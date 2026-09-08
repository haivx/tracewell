# Project Research Summary

**Project:** Tracewell — A Minimum Version of Braintrust  
**Domain:** TypeScript agent observability, evaluation, and checkpoint/recovery infrastructure (portfolio / learning project)  
**Researched:** 2026-09-08  
**Confidence:** HIGH (stack, features, architecture well-documented; checkpoint/recovery design has clear precedent; pitfalls grounded in 2025-2026 practitioner research)

---

## Executive Summary

Tracewell is a self-built, minimum-viable implementation of Braintrust's core infra layer (tracing, evaluation, checkpoint/recovery) around a single hand-written code-review agent. The project's stated value is a live demo ("bug appears, trace explains why, eval catches the regression") backed by a defensible architecture teardown — this is explicitly a portfolio and learning piece, not a product to scale.

**Research confirms the recommended approach:** build the agent loop (no framework), build the trace schema (own the mechanism), build the eval harness (own the learning), adopt the Anthropic SDK and database drivers (zero learning value in reimplementing streaming or SQL layers). The hidden complexity is not in the stack — all choices are well-established — but in three process and design risks: (1) the quality gate is noise-dominated at this dataset size and requires explicit noise-floor-aware design, not a naive scalar threshold; (2) the trace schema is missing error/exception capture, which the demo's core value prop requires; (3) score aggregation (how hard assertions + judge combine into one run-level number) is assumed but never decided, and is load-bearing for both the quality gate and run comparison.

**Biggest risk:** the project stalls before the architecture teardown is written. This is explicitly named in PROJECT.md as the top risk. Starting the teardown in Phase 1 (as a running decision log) and updating it every phase thereafter is non-negotiable risk mitigation.

---

## Key Findings

### Recommended Stack

**Core decision:** Build the trace schema and agent loop yourself; adopt everything else. The learning objective is the instrumentation boundary and span semantics — frameworks hide both. OTel's GenAI conventions are still experimental (Development stability, no timeline to Stable), so borrow the naming (`gen_ai.*` attributes) but not the full SDK (which adds indirection over the exact mechanism the project teaches).

**Build:**
- Agent loop (manual `while` loop, no `toolRunner()` SDK helper)
- Trace span schema (simple: `id`, `trace_id`, `parent_span_id`, `name`, `kind`, `status`, `start_time`, `end_time`, `attributes` JSON, `input_tokens`, `output_tokens`, `cost_usd`)
- Eval harness (dataset loader + task runner + scorers + comparison table — not Evalite/promptfoo/vitest-evals)
- Cost math (static pricing table + token counts from SDK usage fields)
- Quality gate logic (noise-floor-aware, not scalar threshold)
- Checkpoint/recovery (durability-controlled, separate from span writes)

**Adopt:**
- `@anthropic-ai/sdk` (0.124.0) for streaming, structured outputs, and usage fields
- `better-sqlite3` (13.0.3) for local store; mature sync API matters for checkpoint durability
- `drizzle-orm` (0.45.2) for schema parity across SQLite→Postgres migration
- `zod` (v4) for schema validation and structured-output parsing
- `vitest` for unit testing the codebase (NOT as the eval pipeline)

### Expected Features

**Table stakes (must exist for demo to land):**
- Trace capture with spans + parent-child nesting
- Span waterfall UI with recursive tree render and timeline
- Tool call inputs/outputs per span (so trace explains the bug)
- Token + cost per span, rolled up to parent
- **Error surfacing on the failing span** — MISSING from stated requirements but load-bearing for core value
- Hard assertions (lint/test pass-fail) + LLM-as-judge scorers
- Run-over-run comparison / diff view
- **Score aggregation formula** — MISSING from stated requirements but prerequisite for comparison + gate
- Quality gate (warn on score threshold regression)

**Differentiators (what makes this worth showing an interviewer):**
- **Checkpoint/recovery for durable execution** — Braintrust explicitly does NOT do this. This project's read-only tools make it tractable.
- Trace → eval-case conversion (optional stretch goal)
- Integrated cost/token breakdown in dashboard

**Explicitly NOT built:** Multi-tenancy, RBAC, sampling, columnar store, prompt playground, annotation queues, trend dashboard, statistical significance testing

### Architecture Approach

**Single most important principle:** `agent/` has **zero imports** from `tracing/` or `checkpoint/`. Instrumentation wraps from outside via AsyncLocalStorage context + HOF wrapping at two seams (model call, tool call).

**Major components:**

1. **Agent loop** — Plan/act/observe, tool calling, no framework awareness of tracing or checkpointing
2. **Tracing layer** — AsyncLocalStorage context; span tree via id/parent_span_id/trace_id; buffered writes to SQLite
3. **Trace store** — `runs`/`spans` tables queryable per run; supports nesting, latency, tokens, cost
4. **Eval harness** — Dataset loader, task runner (same instrumented agent as live CLI), hard + LLM judge scorers, comparison table
5. **Checkpoint/recovery** — Separate `checkpoints`/`tool_results` tables with synchronous write-ahead durability; resume via message reconstruction + idempotent tool replay
6. **Quality gate** — Noise-floor-aware threshold compare (hard assertions gate strictly; judge score drift is warning)
7. **Dashboard** — Next.js Server Components, trace waterfall, run diff, cost breakdown

### Critical Pitfalls to Avoid

1. **Lost parent-child trace context under concurrency** — Parallel tool calls must explicitly share parent span context; test immediately in Phase 1
2. **LLM-judge unreliability at small dataset** — Position bias, self-preference, verbosity bias, non-determinism. With 10-20 cases, judge variance alone (±15-26 points) dominates signal. Calibrate against hand-labels first.
3. **Quality gate as naive scalar threshold** — At this noise floor, a simple "fail if score < 80%" is noise-dominated. Correct: hard-assertion regressions gate strictly, judge drift is informational.
4. **Spans not closed on error paths** — Exceptions skip the `finally` that closes spans. Use single `withSpan()` helper enforcing try/catch/finally; test forced exceptions early.
5. **Process risk: Stalling before teardown is written** — Explicitly named as top project risk. Mitigate by starting teardown in Phase 1 as running log, updating every phase.

---

## Implications for Roadmap

### Suggested Phase Structure (Validated Against Dependencies)

**Phase 1: Agent Loop + Tracing**
- **Rationale:** Both must exist together; tracing wraps agent seams to establish instrumentation boundary
- **Delivers:** Hand-written agent + trace schema (spans, nesting, latency, tokens, cost, **error status**) + automatic trace capture
- **Key design clarity needed:** Error/exception field in trace schema; versioned, dated pricing constant
- **Tests:** Error-path span closure (force tool throw); concurrency (parallel tools share parent); attribute truncation (large payloads marked)

**Phase 2: Eval Harness (+ Quality Gate)**
- **Rationale:** Depends on tracing; lower-risk than checkpoint/recovery; produces demoable artifact (comparison table) fast
- **Delivers:** Fixture repo (10-20 seeded bugs) + task runner (reuses instrumented agent) + hard + LLM judge scorers + comparison table + **quality gate as final step**
- **Key design clarity needed:** Score aggregation formula (weighted average? separate metrics?); baseline run designation; noise-floor reporting
- **Tests:** Judge calibration (hand-label subset, measure agreement); judge sanity checks (empty, wrong, unknown inputs); noise-floor reporting in output

**Phase 3: Checkpoint/Recovery**
- **Rationale:** Sequenced after eval for morale (working eval exists before tackling hardest problem), but no hard dependency on it
- **Delivers:** Checkpoint/tool_results tables + resume CLI + trace continuity across crash + idempotent replay
- **Complexity reality:** Read-only agent tools make this tractable vs. general case
- **Tests:** Mid-tool-call crash recovery (not just between turns); checkpoint round-trip via JSON; tool replay idempotency

**Phase 4: Dashboard + Demo + Teardown + Supabase Migration**
- **Rationale:** Reads all prior tables; SQLite→Supabase migration point
- **Delivers:** Trace waterfall UI + run comparison diff + cost breakdown + prepared demo case (rehearsed 3-5 times, recorded fallback) + architecture teardown
- **Scope discipline:** Build only 3 dashboard views (waterfall, diff, cost); no trend dashboard, no extra filters; limit dashboard work to 2-3 sessions; teardown must start before dashboard is "finished"
- **Teardown:** Contrast against Braintrust/LangSmith/Langfuse feature-by-feature; explain noise-floor-aware gate; justify why checkpoint/recovery matters; scope statement on what changes at scale

### Cross-Phase Insights

- **Quality gate does NOT depend on checkpoint/recovery.** The gate (threshold compare over eval output) is independent. Both were originally grouped in one "Phase 3" but research validates the project's later decision to split them. Recommend building gate as final step of Phase 2 (eval), letting checkpoint/recovery stand alone as its own phase.

- **Checkpoint/recovery and quality gate are independent sequencing.** If roadmap later adds an eval case testing "resume produces equivalent score," that case would depend on Phase 3 existing. But the gate itself does not.

### Research Flags (Deeper Planning-Phase Research Needed)

| Phase | Flag | Why |
|-------|------|-----|
| Phase 1 | Trace schema error capture + cost computation pattern | Design clarity: error/status fields + dated pricing constant are cited in research but not yet formalized in PROJECT.md |
| Phase 2 | Score aggregation formula + baseline designation | Must be explicit design decision (weighted average? separate metrics?), not ad-hoc during code |
| Phase 2 | LLM-judge calibration if initial rubric < ~90% agreement | Prepare to iterate judge prompt based on hand-label disagreements |
| Phase 3 | Dangling tool-call cleanup on crash (edge case) | Requires careful handling; refer to durable-execution research; test mid-tool-call crashes explicitly |
| Phase 4 | None — standard patterns apply | Next.js/Drizzle/Supabase are all well-documented |

### Standard Patterns (Skip Research-Phase)

- **Phase 4 Dashboard:** Next.js Server Components, Drizzle ORM for Postgres, SQLite→Postgres migration — all well-established with production examples
- **Phase 1 AsyncLocalStorage:** Node.js async context tracking is settled in official docs and OTel reference implementations

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| **Stack** | HIGH | Versions verified via Context7/npm (2026-09-08); build-vs-adopt grounded in learning objectives |
| **Features** | HIGH | Unanimous across 6 vendors (Braintrust, LangSmith, Langfuse, Arize, Weave, Laminar); gaps identified vs. stated core value |
| **Architecture** | HIGH (tracing/eval) / MEDIUM (checkpoint specifics) | Patterns are industry-standard; checkpoint less standardized but has LangGraph precedent |
| **Pitfalls** | HIGH | Sourced from 2025-2026 research (ACL/EMNLP on judge bias, Arize/Diagrid on agent failures, Anthropic eval checklist) |

**Overall: HIGH** — Stack is clear. Features validated. Architecture is established. Pitfalls grounded in recent research.

### Gaps to Address During Planning

1. **Trace schema error field** — Add explicit `status` (ok/error) and `error` text fields; load-bearing for "trace explains why" demo
2. **Score aggregation formula** — Make this an explicit Phase 2 design decision (hard assertions + LLM judge → run-level score); not ad-hoc
3. **Baseline run designation** — Clarify which run in a comparison is "reference" for regression directionality
4. **Checkpoint idempotency semantics** — Document the assumption "re-execute any uncompleted tool on resume is safe because tools are read-only"
5. **Teardown authorship schedule** — Start in Phase 1 as running log; commit to one paragraph per phase as part of each phase's definition of done
6. **Demo rehearsal schedule** — Rehearse 3-5 times before recording/presenting live; fallback recorded version required

---

## Sources

**Primary (HIGH confidence):**
- STACK.md, FEATURES.md, ARCHITECTURE.md, PITFALLS.md (2026-09-08, Context7-verified and cross-referenced)
- PROJECT.md (project constraints)
- ACL 2025: Judging the Judges — Position Bias in LLM-as-a-Judge
- EMNLP 2025: Self-Preference Bias in LLM-as-a-Judge
- Arize, Diagrid, Medium: AI agent failure modes and checkpoint/replay pitfalls
- Node.js official docs: AsyncLocalStorage context propagation
- Anthropic eval-audit checklist: noise-floor formula, judge reliability

**Secondary (MEDIUM confidence):**
- Braintrust/LangSmith/Langfuse vendor documentation and blog comparisons
- LangGraph Persistence documentation (checkpoint design precedent)
- Zylos Research: Durable execution for AI agents

---

*Research synthesis completed: 2026-09-08*  
*Status: Ready for roadmap planning*
