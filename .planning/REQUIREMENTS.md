# Requirements: Tracewell

**Defined:** 2026-09-08
**Core Value:** A live, screen-shareable demo of the full loop — a bug appears, the trace explains why, the eval catches the regression — backed by an architecture teardown the author can defend in depth.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Agent

- [ ] **AGENT-01**: Agent runs a tool-calling loop (model decides → tool executes → result returns → repeat) until it produces a final answer
- [ ] **AGENT-02**: Agent exposes four tools to the model: `read_file`, `run_linter`, `run_tests`, `search_codebase`
- [ ] **AGENT-03**: User can run the agent against a target repo from a CLI command
- [ ] **AGENT-04**: Agent handles multiple tool calls requested in a single model turn
- [ ] **AGENT-05**: Agent emits its review findings as Zod-validated structured output, not prose

### Tracing

- [ ] **TRACE-01**: Trace schema stores runs and spans, with spans forming a tree via `parent_span_id`
- [ ] **TRACE-02**: Spans carry a kind distinguishing agent turns, LLM calls, and tool calls
- [ ] **TRACE-03**: Every agent run auto-logs a complete trace without the agent importing from the tracing module
- [ ] **TRACE-04**: Spans record status and capture errors, including on failure paths
- [ ] **TRACE-05**: Spans record token usage and computed cost against a dated pricing table
- [ ] **TRACE-06**: Span attribute payloads are byte-capped so stored prompts and file contents cannot bloat the store
- [ ] **TRACE-07**: A completed run is queryable as one full trace

### Eval

- [ ] **EVAL-01**: A fixture repo of seeded bugs provides deterministic ground truth for 10-20 eval cases
- [ ] **EVAL-02**: Eval cases run through the same instrumented agent entrypoint as live runs, so each case emits a real trace
- [ ] **EVAL-03**: Hard assertion scorers check deterministic outcomes against seeded ground truth
- [ ] **EVAL-04**: An LLM-as-judge scorer produces structured, schema-validated scores
- [ ] **EVAL-05**: The LLM judge is calibrated against hand-labelled cases before the gate relies on it
- [ ] **EVAL-06**: An explicit, documented formula aggregates assertion and judge scores into one run-level number
- [ ] **EVAL-07**: User can run an eval suite from a CLI command
- [ ] **EVAL-08**: User can compare two named runs in a table showing which cases changed verdict

### Quality Gate

- [ ] **GATE-01**: The gate blocks on hard-assertion regressions
- [ ] **GATE-02**: The gate reports judge-score drift as informational within a computed noise-floor band, rather than failing on a scalar threshold
- [ ] **GATE-03**: The measured noise floor for the current dataset size is computed and reported

### Recovery

- [ ] **RECOV-01**: Agent state (message history, tool results, step index) is checkpointed at tool-result boundaries, written before the next model call
- [ ] **RECOV-02**: User can resume a failed run from its last checkpoint instead of restarting it
- [ ] **RECOV-03**: Tool results are keyed by `tool_call_id` so replay on resume is idempotent
- [ ] **RECOV-04**: A resumed run produces a trace that remains coherent across the resume boundary

### Dashboard

- [ ] **DASH-01**: User can view a trace as a waterfall showing span nesting, latency, tool inputs/outputs, and errors
- [ ] **DASH-02**: User can view two runs side by side, showing which cases changed verdict
- [ ] **DASH-03**: User can view cost and token breakdown per run, per model, and per tool
- [ ] **DASH-04**: The trace store is migrated from SQLite to Supabase and the dashboard reads from it

### Artifacts

- [ ] **ART-01**: The architecture teardown is started in the tracing phase and appended as a deliverable of every subsequent phase
- [ ] **ART-02**: The architecture teardown is complete and defensible at project close
- [ ] **ART-03**: A demo script walks bug → trace → eval-catches-regression, rehearsed until it runs reliably
- [ ] **ART-04**: A recorded demo exists as a fallback for when a live model call misbehaves
- [ ] **ART-05**: A README documents setup and how to re-run the project after time away

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Agent

- **AGENT-06**: Graceful handling of `max_tokens` truncation mid-tool-use
- **AGENT-07**: Streaming output from the agent CLI

### Eval

- **EVAL-09**: Eval score trend over time across many runs
- **EVAL-10**: Promotion of an interesting live trace into a new eval case
- **EVAL-11**: A "crash and resume produces an equivalent score" eval case

### Tracing

- **TRACE-08**: An OpenTelemetry-compatible exporter, as a "how this would scale" appendix to the teardown

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| High-performance trace store (Brainstore-class) | Irrelevant at this scale; the learning is in the schema, not the storage engine |
| Multi-tenancy, RBAC, SOC2, compliance | No external users |
| Support for multiple agent frameworks | One hand-built agent is the whole subject |
| Agent frameworks for the agent itself (Vercel AI SDK, Mastra) | Understanding the tool-calling loop at a primitive level is the point |
| The SDK's beta `toolRunner()` helper | Automates precisely the loop this project exists to teach |
| Full OpenTelemetry SDK and Collector pipeline | GenAI semconv still at Development stability; overkill for one process, one language. Borrow `gen_ai.*` naming only |
| Eval libraries (Evalite, promptfoo, vitest-evals) | Not agent frameworks, and technically sound — but the eval harness is itself the thing being learned |
| Full prompt/agent versioning model | Braintrust's product surface, not the learning objective |
| Session / thread grouping | This agent produces one trace per run, not multi-turn chat sessions |
| Sampling, retention policies, trace ingestion at scale | No volume to sample |
| Public-repo-grade onboarding | The finish line is a screen-shareable demo, not a project others clone |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| (populated during roadmap creation) | | |

**Coverage:**
- v1 requirements: 36 total
- Mapped to phases: 0
- Unmapped: 36 ⚠️

---
*Requirements defined: 2026-09-08*
*Last updated: 2026-09-08 after initial definition*
