# Roadmap: Tracewell

## Overview

Tracewell builds a minimum version of Braintrust's infra layer around one hand-written code-review
agent. The journey runs bottom-up along a hard dependency chain: first a standalone agent that runs
a real tool-calling loop from the CLI, then a tracing layer that wraps it from the outside without
the agent ever knowing, then an eval harness whose cases run through that same instrumented
entrypoint and a quality gate honest enough to survive a small dataset's noise floor, then
checkpoint/recovery — the hardest phase, deliberately sequenced after a working eval loop is banked —
and finally a Next.js dashboard that makes all of it visible on a screen share, plus the rehearsed
demo and the finished architecture teardown.

Two structural commitments shape everything: the agent has **zero imports** from the tracing module
(proved by construction, since the agent ships a full phase before tracing exists), and the
architecture teardown is a **running log** started in Phase 2 and appended by every phase after it —
never written cold at the end. Stalling before the teardown is this project's top risk, so it is a
named deliverable in four of five phases rather than a final task.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Agent** - A hand-written code-review agent, runnable from the CLI, with a real tool-calling loop and structured output
- [ ] **Phase 2: Tracing** - Automatic trace capture wrapping the agent from outside, producing one queryable span tree per run
- [ ] **Phase 3: Eval + Quality Gate** - Fixture-backed eval suite with assertion and judge scorers, run comparison, and a noise-floor-aware gate
- [ ] **Phase 4: Checkpoint & Recovery** - Checkpointing at tool-result boundaries so a failed run resumes instead of restarting
- [ ] **Phase 5: Dashboard, Demo & Teardown** - Next.js dashboard on Supabase, the rehearsed live demo, and the finished architecture teardown

## Phase Details

### Phase 1: Agent
**Goal**: A code-review agent the user can point at any repo from the CLI and get structured findings back from a real model → tool → result → repeat loop
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: AGENT-01, AGENT-02, AGENT-03, AGENT-04, AGENT-05
**Success Criteria** (what must be TRUE):
  1. User runs a single CLI command against a target repo path and gets back review findings as Zod-validated structured data, not prose
  2. The agent completes a full tool-calling loop unaided — the model requests `read_file`, `run_linter`, `run_tests`, or `search_codebase`, the tool executes, the result returns, and the loop continues until the model produces a final answer
  3. When the model requests several tools in one turn, every one of them executes and every result returns together in the next turn
  4. Pointed at a repo containing a known defect, the agent surfaces that defect in its findings
**Plans**: TBD

### Phase 2: Tracing
**Goal**: Every agent run automatically produces one complete, queryable trace explaining what happened — without the agent containing a single line of tracing code
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: TRACE-01, TRACE-02, TRACE-03, TRACE-04, TRACE-05, TRACE-06, TRACE-07, ART-01
**Success Criteria** (what must be TRUE):
  1. Running the Phase 1 CLI command completely unchanged produces exactly one trace, retrievable afterwards as a single complete run
  2. That trace reads as a span tree — agent turns, LLM calls, and tool calls nested by parent — carrying latency on every span, and token usage plus cost computed against a dated pricing table on every billable one, with oversized prompt and file payloads visibly truncated rather than stored whole
  3. A deliberately failed tool still leaves a closed span carrying error status and the error text, and the run is marked failed rather than left hanging open
  4. A turn issuing parallel tool calls produces sibling spans that all share the correct parent turn — no orphans, no misattribution
  5. `agent/` contains zero imports from `tracing/` (grep-verifiable), and the agent still runs correctly with tracing switched off
  6. The architecture teardown exists as a running document, opened with its first entry: the trace schema and the instrumentation-boundary decision
**Plans**: TBD

### Phase 3: Eval + Quality Gate
**Goal**: A repeatable eval suite over seeded-bug fixtures that scores every run and tells the user whether a change is a real regression or just noise
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: EVAL-01, EVAL-02, EVAL-03, EVAL-04, EVAL-05, EVAL-06, EVAL-07, EVAL-08, GATE-01, GATE-02, GATE-03
**Success Criteria** (what must be TRUE):
  1. User runs one CLI command and 10-20 fixture cases execute end to end, each one emitting a real trace through the same instrumented agent entrypoint a live run uses — not a parallel eval-only code path
  2. Each case yields a hard-assertion score against seeded ground truth plus a schema-validated LLM-judge score, aggregated into one run-level number by a formula written down before it is coded
  3. The judge's agreement against a hand-labelled subset is measured and reported before the gate is allowed to rely on it, and the measured noise floor for the current dataset size is printed with every eval run
  4. User names two runs and gets a table showing which cases changed verdict and in which direction
  5. The gate fails the run on any hard-assertion regression, and reports judge-score movement as informational against the computed noise-floor band — it never fails on a scalar score threshold alone
**Plans**: TBD

*Teardown deliverable: append the scoring-aggregation formula, the judge calibration result, and the noise-floor-aware gate rationale.*

### Phase 4: Checkpoint & Recovery
**Goal**: A run that dies mid-flight resumes from its last completed tool result instead of starting over
**Mode:** mvp
**Depends on**: Phase 2 (Phase 3 is a sequencing choice, not a dependency)
**Requirements**: RECOV-01, RECOV-02, RECOV-03, RECOV-04
**Success Criteria** (what must be TRUE):
  1. User kills a run mid-flight, issues a resume command, and the run continues from its last completed tool result rather than replaying from the first model call
  2. Agent state — message history, tool results, step index — is written at tool-result boundaries (after the tool returns, before the next model call) and survives a hard process kill
  3. A resumed run does not re-execute tools that already completed; their results are recovered by `tool_call_id` so replay is idempotent
  4. The trace for a crashed-then-resumed run reads as one coherent trace across the resume boundary, showing plainly where the break happened
  5. The teardown has gained its checkpoint entry, including why buffered span writes and synchronous checkpoint writes get different durability policies, and why read-only tools make idempotent replay safe
**Plans**: TBD

### Phase 5: Dashboard, Demo & Teardown
**Goal**: The whole loop is visible on a screen share and rehearsed until reliable — a bug appears, the waterfall explains why, the eval catches the regression — backed by a teardown the author can defend in depth
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: DASH-01, DASH-02, DASH-03, DASH-04, ART-02, ART-03, ART-04, ART-05
**Success Criteria** (what must be TRUE):
  1. User opens the dashboard — reading from Supabase, with the existing SQLite traces migrated across and nothing lost — and views any run as a waterfall showing span nesting, latency, tool inputs and outputs, and the error on the failing span
  2. User views two named runs side by side and sees which cases changed verdict
  3. User views a cost and token breakdown for a run, sliced per model and per tool
  4. The author runs the demo front to back — bug → trace → eval catches the regression — reliably enough to do it live, with a recorded version standing by as a fallback
  5. The architecture teardown is complete and defensible end to end, and a README gets the author from a cold checkout to a re-run after time away
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Agent | 0/TBD | Not started | - |
| 2. Tracing | 0/TBD | Not started | - |
| 3. Eval + Quality Gate | 0/TBD | Not started | - |
| 4. Checkpoint & Recovery | 0/TBD | Not started | - |
| 5. Dashboard, Demo & Teardown | 0/TBD | Not started | - |

## Coverage

All 36 v1 requirements are mapped to exactly one phase each.

| Phase | Requirements | Count |
|-------|--------------|-------|
| 1. Agent | AGENT-01 … AGENT-05 | 5 |
| 2. Tracing | TRACE-01 … TRACE-07, ART-01 | 8 |
| 3. Eval + Quality Gate | EVAL-01 … EVAL-08, GATE-01 … GATE-03 | 11 |
| 4. Checkpoint & Recovery | RECOV-01 … RECOV-04 | 4 |
| 5. Dashboard, Demo & Teardown | DASH-01 … DASH-04, ART-02 … ART-05 | 8 |
| **Total** | | **36** |

**Note on ART-01:** the architecture teardown is a running log by design. It is *owned* by Phase 2,
where it starts, and appended as an explicit deliverable of Phases 3, 4, and 5. ART-02 (teardown
complete and defensible) closes it out in Phase 5.

---
*Roadmap created: 2026-09-08*
