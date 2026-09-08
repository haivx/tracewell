# Architecture Research

**Domain:** Agent observability + eval + checkpoint/recovery infrastructure (bespoke "mini-Braintrust")
**Researched:** 2026-09-08
**Confidence:** HIGH (data model, instrumentation, write-path, dashboard) / MEDIUM (checkpoint/recovery specifics, since this is the least standardized area and Tracewell's design will be somewhat original)

## Standard Architecture

### System Overview

```
┌───────────────────────────────────────────────────────────────────────┐
│                         Agent Runtime (Phase 0)                       │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐            │
│  │ Agent Loop     │──▶│ Claude API     │   │ Tool Executors │           │
│  │ (plan/act/obs) │◀──│ (messages)     │   │ (fs, lint,     │◀──┐       │
│  └───────┬────────┘   └───────────────┘   │  test, search) │  │       │
│          │                                └───────┬────────┘  │       │
│          │  wrapped by (no code changes to loop)  │           │       │
├──────────┼────────────────────────────────────────┼───────────┼───────┤
│          ▼           Tracing Layer (Phase 1)      ▼           │       │
│  ┌────────────────────────────────────────────────────────┐   │       │
│  │ withSpan()/AsyncLocalStorage context  →  Span buffer    │───┘       │
│  │  span.start/end, attributes, usage/cost attach          │           │
│  └───────────────────────┬──────────────────────────────────┘         │
├──────────────────────────┼─────────────────────────────────────────────┤
│                           ▼        Checkpoint Layer (Phase 3)          │
│  ┌────────────────────────────────────────────────────────┐           │
│  │ Step log: message history + tool results + step index    │           │
│  │ Written BEFORE each risky action; resumable run loader    │           │
│  └───────────────────────┬──────────────────────────────────┘           │
├──────────────────────────┼─────────────────────────────────────────────┤
│                           ▼         Storage (SQLite → Supabase)        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐             │
│  │ runs     │   │ spans    │   │ checkpoints│  │ eval_*   │             │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘             │
├─────────────────────────────────────────────────────────────────────────┤
│                    Eval Harness (Phase 2)     Quality Gate (Phase 3.5)  │
│  ┌────────────────┐ ┌────────────┐ ┌───────────┐ ┌───────────────────┐│
│  │ Dataset (cases) │▶│ Task runner│▶│ Scorers   │▶│ threshold compare ││
│  │ (fixture repo)  │ │ (runs agent)│ │ (assert+  │ │ across run A vs B││
│  └────────────────┘ └────────────┘ │ LLM-judge)│ └───────────────────┘│
├─────────────────────────────────────────────────────────────────────────┤
│                         Dashboard (Phase 4, Next.js)                    │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌──────────────────┐│
│  │ RSC: trace waterfall │  │ RSC: run comparison │  │ RSC: cost/token  ││
│  │ (direct DB query)    │  │ diff                │  │ breakdown        ││
│  └─────────────────────┘  └─────────────────────┘  └──────────────────┘│
└───────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|-------------------------|
| Agent loop | Decide next action, call model, call tools, repeat until done/error | Plain async function, no tracing awareness — sees only `messages[]`, `tools[]` |
| Tracing SDK | Create/close spans, propagate parent context implicitly, attach usage/cost, buffer + flush | `AsyncLocalStorage`-backed context + a `withSpan()` wrapper around the two "seams" (model call, tool call) |
| Trace store | Durable, queryable record of every span ever produced | SQLite table (`spans`), later a Supabase/Postgres table with same shape |
| Checkpoint store | Durable, resumable record of *agent execution state* (not just what happened, but what to do next) | Either same SQLite DB, different table (`checkpoints`), or derived view over spans — decide explicitly (see recovery section) |
| Eval harness | Load dataset, run task under trace, score output, aggregate | `runEval(dataset, task, scorers)` — a CLI command producing a comparison table |
| Quality gate | Compare eval run against baseline/threshold, exit non-zero on regression | Thin wrapper over eval harness output; CI-shaped, not a new subsystem |
| Dashboard (Next.js) | Query stores, render trace waterfall / run diff / cost breakdown | Server Components query the DB directly; no API layer needed until there's a client-side interactive feature (filtering, live tailing) |

## Recommended Project Structure

```
src/
├── agent/                      # Phase 0 — has ZERO imports from tracing/
│   ├── loop.ts                 # the plan→act→observe loop
│   ├── tools/                  # read_file, run_linter, run_tests, search_codebase
│   └── client.ts                # thin Claude API wrapper
├── tracing/                     # Phase 1 — wraps agent from OUTSIDE
│   ├── context.ts                # AsyncLocalStorage store, currentSpan()
│   ├── span.ts                   # startSpan/endSpan, span tree building
│   ├── instrument.ts              # withSpan() HOF, instrumentModel(), instrumentTool()
│   ├── writer.ts                  # buffered writer → SQLite
│   └── schema.sql                 # runs, spans tables
├── eval/                         # Phase 2 — depends on agent + tracing
│   ├── dataset.ts                 # load fixture cases
│   ├── runner.ts                   # runs task per case, opens a trace per case
│   ├── scorers/                    # exact-match/assert scorers + llm-judge scorer
│   └── report.ts                    # comparison table across two run ids
├── checkpoint/                    # Phase 3 — depends on agent + tracing
│   ├── store.ts                     # persistCheckpoint/loadCheckpoint
│   ├── resumable-loop.ts            # wraps agent/loop.ts with checkpoint boundaries
│   └── schema.sql                    # checkpoints table (or view over spans)
├── gate/                          # Phase 3.5 — depends on eval only
│   └── check-threshold.ts
├── db/
│   ├── sqlite.ts                    # better-sqlite3 connection, migrations
│   └── supabase.ts                  # Phase 4: Postgres client behind same query interface
└── dashboard/                       # Phase 4 — Next.js app, reads db/ only
    ├── app/
    │   ├── runs/[id]/page.tsx        # server component, direct query
    │   ├── compare/page.tsx
    │   └── api/                       # only for anything client-interactive (search, live refresh)
    └── components/
        ├── waterfall.tsx
        └── diff-table.tsx
```

### Structure Rationale

- **`agent/` has no dependency on `tracing/`.** This is the single most important structural rule in this project — it's the whole point of "instrumentation boundary" (question 2). If `agent/loop.ts` imports anything from `tracing/`, the boundary has failed.
- **`tracing/` and `checkpoint/` are siblings, not one-inside-the-other**, because they answer different questions (what happened vs. what to do next) even though they may share a physical SQLite file. Keeping them as separate modules with separate schemas keeps the Phase 3 checkpoint work from becoming "add more columns to spans," which is a trap (see Anti-Patterns).
- **`db/` is a thin swappable layer** so the SQLite→Supabase migration at the dashboard phase touches one file's implementation, not every call site — this is the concrete mechanism behind the "migration boundary" the project stack decision expects.
- **`eval/` sits beside `checkpoint/`, both depending on `agent/` + `tracing/`**, which is why they can be built in either order relative to each other, but both need tracing to exist first (see Build Order).

## Architectural Patterns

### Pattern 1: Span tree via trace_id / span_id / parent_span_id (not OTel's full model)

**What:** Every unit of work — one agent run, one model call, one tool call — is a row with `id`, `trace_id` (shared by the whole run), `parent_span_id` (nullable, null = root), `name`, `kind`, `start_time`, `end_time`, `status`, `attributes` (JSON blob), and for LLM spans specifically `input_tokens`, `output_tokens`, `cost_usd`.

**When to use:** This is the right level of complexity for a bespoke single-process TypeScript system. Full OpenTelemetry (Resource/Scope/Span protobuf, W3C traceparent propagation, exporters, collectors) is built for distributed, multi-service, multi-vendor environments — none of which applies here (one process, one language, one store).

**Trade-offs:**
- OTel's GenAI semantic conventions (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`, span kinds `chat` / `invoke_agent` / `execute_tool`) are worth **stealing the attribute names from**, even if you don't adopt the OTel SDK. Doing so means the teardown document can honestly say "this maps onto the industry-standard GenAI semconv," which is a real interview-credibility point, at zero implementation cost — it's just naming discipline.
- Langfuse's actual data model is the closer structural analog: a **trace** is a request-level grouping (`trace_id`), and **observations** (their term for span-like units) form a tree via `parent_observation_id`, with three observation types — `SPAN` (generic), `GENERATION` (LLM call, carries model/usage/cost), and `EVENT` (point-in-time). This three-type split (generic span / LLM generation / point event) is a better fit for Tracewell than OTel's generic-span-plus-attributes model, because it gives the dashboard an immediate, cheap way to distinguish "this row is billable and has tokens" from "this row is just structural nesting" without attribute-sniffing.
- Braintrust's model is span-centric too: `span_id`, `root_span_id`, and `span_parents` (plural — Braintrust spans can have multiple logical parents for cross-cutting concerns) are managed by the SDK, and a `span_attributes.type` field (`llm | score | function | eval | task | tool | review`) drives UI rendering. The multi-parent design is overkill for Tracewell — a single `parent_span_id` (strict tree, not DAG) is sufficient and much simpler to render as a waterfall.

**Recommendation for Tracewell — a hybrid, adopt this schema:**

```sql
CREATE TABLE runs (
  id            TEXT PRIMARY KEY,       -- trace_id, one per agent invocation
  kind          TEXT NOT NULL,          -- 'agent_run' | 'eval_case'
  status        TEXT NOT NULL,          -- 'running' | 'completed' | 'failed'
  started_at    INTEGER NOT NULL,
  ended_at      INTEGER,
  metadata      TEXT                    -- JSON: eval_case_id, dataset_version, agent_version, etc.
);

CREATE TABLE spans (
  id              TEXT PRIMARY KEY,
  run_id          TEXT NOT NULL REFERENCES runs(id),
  parent_span_id  TEXT REFERENCES spans(id),  -- NULL = root span of the run
  kind            TEXT NOT NULL,   -- 'agent_turn' | 'llm_call' | 'tool_call'
  name            TEXT NOT NULL,   -- e.g. 'run_linter', 'claude.messages.create'
  status          TEXT NOT NULL,   -- 'ok' | 'error'
  started_at      INTEGER NOT NULL, -- epoch ms
  ended_at        INTEGER,
  input           TEXT,             -- JSON, redact/truncate large payloads
  output          TEXT,             -- JSON
  error           TEXT,
  attributes      TEXT,             -- JSON: tool name/args, model params, etc.
  input_tokens    INTEGER,          -- only set on kind='llm_call'
  output_tokens   INTEGER,
  cost_usd        REAL
);
CREATE INDEX idx_spans_run ON spans(run_id);
CREATE INDEX idx_spans_parent ON spans(parent_span_id);
```

Cost attaches at write time: the `llm_call` span wrapper knows the model name and the token counts returned by the Claude API response, and looks up a static price table (`$/1K input`, `$/1K output`, updated by hand when pricing changes) to compute `cost_usd` before the span is closed. Do not defer cost computation to the dashboard — compute it once, at the source, so historical spans stay correct even if prices change later.

### Pattern 2: AsyncLocalStorage as the implicit-context primitive, with explicit "seam" wrapping

**What:** `AsyncLocalStorage<Span>` holds "the currently active span" for whatever async call chain is executing. A `withSpan(name, kind, fn)` helper creates a child span (parent = whatever's currently in the store), runs `fn` inside `als.run(newSpan, fn)`, and closes the span in a `finally`. Because `AsyncLocalStorage` propagates across `await`, `Promise.then`, `setTimeout`, etc., any code called inside `fn` — even several `await`s deep — can call `currentSpan()` and get the right parent without having a span object threaded through every function signature.

**When to use:** This is the correct primitive for Tracewell, and it is exactly what OpenTelemetry's own Node SDK uses internally (`AsyncHooksContextManager`/`AsyncLocalStorageContextManager`) to make context propagation "invisible" to instrumented code. Concretely for this project:

- **Do NOT decorate the agent loop's business logic.** The agent loop should stay exactly as it would be if tracing didn't exist: `const response = await client.messages.create(...)`, `const result = await tool.run(args)`.
- **Wrap only the two seams that cross a system boundary**: the Claude API call, and each tool's `run()`. This is the entire instrumentation surface. Concretely:
  ```typescript
  // tracing/instrument.ts
  export function instrumentModelCall(client: Anthropic) {
    const original = client.messages.create.bind(client.messages);
    client.messages.create = (async (params: any) => {
      return withSpan('llm_call', 'llm_call', async (span) => {
        span.setAttribute('model', params.model);
        const res = await original(params);
        span.setUsage(res.usage.input_tokens, res.usage.output_tokens);
        return res;
      });
    }) as any;
    return client;
  }

  export function instrumentTool<T extends (...a: any[]) => Promise<any>>(name: string, fn: T): T {
    return (async (...args: any[]) => {
      return withSpan(name, 'tool_call', async (span) => {
        span.setAttribute('args', args);
        return fn(...args);
      });
    }) as T;
  }
  ```
  This is **higher-order function wrapping**, applied at the call site where tools are registered / the client is constructed — not decorators (TS decorators are a worse fit here: they require classes or experimental stage-3 syntax and buy nothing over a plain HOF for wrapping standalone async functions), and not manual "start span / end span" calls sprinkled through the agent loop (that *is* tracing-awareness leaking into business logic, which is the thing to avoid).
- **The agent-turn span** (one per plan/act/observe iteration) is the one place a very thin explicit call is justified, because "a turn" is a concept the tracer knows about but the agent loop's control flow already expresses naturally as a loop iteration — wrap the loop body: `for (...) { await withSpan('agent_turn', 'agent_turn', () => step()) }`. This one call at the loop's own boundary is acceptable and is different from wrapping internals.

**Trade-offs:**
- AsyncLocalStorage has a measured ~7% throughput overhead in Node — irrelevant for an agent loop dominated by seconds-long LLM round trips, so don't optimize this away.
- The known pitfall: never cache a span/context reference in a long-lived object (a connection pool, a singleton tool registry) and reuse it later — the captured context goes stale. In this codebase that means: tool instances should be instrumented per-invocation via the wrapper shown above, not by stashing "the current span" on the tool object itself.
- Decorators were considered and rejected: they add a build-config dependency (experimental decorators or TC39 stage-3 support) for a project that is otherwise plain TS, and provide no capability HOFs don't already give for wrapping async functions.

### Pattern 3: Buffered, run-scoped span writes with a write-ahead checkpoint entry

**What:** Spans are appended to an in-memory array as they close, and flushed to SQLite in a batch (a) periodically (e.g., every N spans or M milliseconds) and (b) always at run completion. This is a buffered write path, not synchronous-per-span, because per-span synchronous SQLite writes on the hot path of an LLM loop add avoidable latency for no benefit when nothing else is reading the data mid-run in this project (no live dashboard tailing is in scope).

**When to use:** Use buffering for **spans** (observability data — losing the last few is a cosmetic gap, not a correctness gap). Do NOT use buffering for **checkpoint state** (recovery data — losing the last write means losing the ability to resume, which is the entire point of Phase 3). This split is the key insight connecting questions 3 and 5: tracing and checkpointing have different durability requirements even though they may share a database file.

Concretely:
- `spans` table: buffered writer, `db.pragma('journal_mode = WAL')`, default `synchronous = NORMAL` is fine (durable across app crashes, theoretically not across OS/power-loss — acceptable for trace data in a portfolio project).
- `checkpoints` table: **synchronous, per-step write**, and consider `db.pragma('synchronous = FULL')` for this table's writes specifically, or at minimum ensure the write is `await`ed and confirmed before the next risky step (a tool call with side effects) is allowed to proceed. Losing a checkpoint write silently is exactly the failure mode Phase 3 exists to prevent.
- On crash mid-run: buffered-but-unflushed spans for that run are lost; this is acceptable and should be a documented, explicit trade-off in the teardown (it doubles as a demonstrable point: "I chose to accept trace data loss on crash but not checkpoint data loss, here's why"). Because `better-sqlite3` is synchronous per call, a `process.on('exit')`/`SIGINT`/`SIGTERM` handler that flushes the span buffer one last time is cheap insurance and worth adding regardless.
- Mark the `runs` row `status = 'crashed'` via the same crash handler (or via a startup sweep that finds `status = 'running'` rows with no matching completion event) so the dashboard can visibly distinguish a crashed run from a completed one — this is also what recovery needs to find resumable runs (see Pattern 5).

**Trade-offs:** Buffering trades a small, bounded amount of trace completeness for write-path simplicity and throughput; this is the correct trade for a trace/observability path and the wrong trade for a recovery path, which is why the two must not share a flush policy even if they share a connection.

### Pattern 4: Eval harness as dataset × task-runner × scorers, where each case IS a trace

**What:** The eval harness is three composable pieces:
1. **Dataset** — an array of cases, each `{ id, input, expected }`, loaded from the fixture repo (e.g., `{ id: 'bug-07', repoPath: 'fixtures/case-07', expectedFindings: ['unused-var:src/foo.ts:12'] }`).
2. **Task runner** — for each case, invokes the agent (the exact same `agent/loop.ts` used in production, not a special "eval mode" of the agent) against the case's input, and captures the result.
3. **Scorers** — pure functions `(case, output, trace) => { name, score, metadata }`, composed of hard assertions (`didFindBug(output, expected)` → 0/1) and an LLM-as-judge scorer (`judgeCodeReviewQuality(output)` → 0–1 via a separate Claude call scored against a rubric).

**How eval runs relate to traces — this must be explicit:** Each eval case execution **is** an agent run and therefore produces exactly one trace, identical in shape to a live/manual invocation. The `runs` table's `kind` column (`'agent_run' | 'eval_case'`) and `metadata` JSON (`eval_id`, `dataset_version`, `case_id`) are what distinguish an eval-produced trace from a manually-triggered one — there is no separate "eval trace" schema. This matters for two reasons: (1) it means the trace viewer built in Phase 1/4 is *automatically* also the eval-run inspector — no separate UI needed; (2) it means a scorer can inspect the trace, not just the final output — e.g., a scorer that checks "did the agent call `run_tests` before concluding," which is impossible without the trace being available at scoring time. Pass the `run_id`/`trace_id` produced by the task runner's invocation into each scorer alongside the raw output.

```typescript
// eval/runner.ts (sketch)
async function runCase(agentFactory: () => Agent, dataset: Case, scorers: Scorer[]) {
  const agent = agentFactory();
  const runId = crypto.randomUUID();
  const output = await withRun(runId, { kind: 'eval_case', metadata: { caseId: dataset.id } },
    () => agent.run(dataset.input));
  const trace = await loadTrace(runId); // read back what tracing/ just wrote
  const scores = await Promise.all(scorers.map(s => s.score({ case: dataset, output, trace })));
  return { runId, caseId: dataset.id, output, scores };
}
```

An eval "run" (the comparison-table-producing unit, e.g. "run all 20 cases against agent v2") is then just a batch of these case-level traces, tagged with a shared `eval_run_id` for grouping in `runs.metadata`. Comparing two named runs (a stated requirement) is a query: `SELECT case_id, score FROM eval_results WHERE eval_run_id IN (?, ?)` pivoted for a diff table — no new storage concept required beyond one more table, `eval_results` (`eval_run_id`, `case_id`, `scorer_name`, `score`, `metadata`).

**Trade-offs:** Running the eval task through the *exact same* instrumented agent (not a stripped-down eval-only path) costs a little more setup (the eval harness needs to open/close runs the same way the CLI entrypoint does) but is what makes "the eval catches the regression, and the trace explains why" — the project's stated demo — actually true. If eval cases didn't produce real traces, the demo would need two different code paths, undermining the "one agent, fully observed" story.

### Pattern 5: Checkpoint/recovery — separate store, write-ahead-of-action semantics, tool-replay via idempotency keys

**This is the hardest phase; treated in depth below, not just as a pattern.**

## Data Flow

### Request Flow (live agent run)

```
CLI invocation
    ↓
agent/loop.ts starts a run  →  tracing: withRun() opens `runs` row (status=running)
    ↓
loop iteration (agent_turn span) ─┬─▶ checkpoint/: persist step boundary (sync write)
                                   │       (message history so far, step index)
    ↓                             │
model call (llm_call span, via instrumented client)
    ↓
tool call (tool_call span, via instrumented tool) ──▶ checkpoint/: persist tool result
    ↓                                                     keyed by (run_id, step_index, tool_call_id)
loop continues or ends
    ↓
tracing: flush span buffer, mark `runs` row status=completed
    ↓
[dashboard reads runs/spans/checkpoints tables — read-only, no write path]
```

### Recovery Flow (resuming a crashed run)

```
`gsd resume <run_id>` (or agent CLI --resume flag)
    ↓
checkpoint/store.ts: loadCheckpoint(run_id)
    ↓
reconstruct: { messages[], lastCompletedStepIndex, pendingToolCalls[] }
    ↓
for each pendingToolCall already recorded as "requested but no result":
    check idempotency key → if a result was actually persisted, reuse it (no re-execution)
    if truly unresolved → re-execute tool call (side-effect risk acknowledged, see below)
    ↓
resume agent/loop.ts from lastCompletedStepIndex + 1, with messages[] pre-seeded
    ↓
tracing: new spans for resumed steps get parent_span_id pointing back into the ORIGINAL
          run's span tree (same run_id) — the trace shows one continuous run with a
          visible gap/marker at the crash point, not two disconnected traces
```

### Key Data Flows

1. **Instrumentation is one-directional and passive:** the agent loop never queries the trace store; tracing only observes and records. This one-directional flow is what keeps `agent/` framework-agnostic and is worth stating explicitly in the teardown as a design principle.
2. **Checkpointing is bidirectional:** unlike tracing, the checkpoint layer both writes (after each step) and reads (at resume time, and potentially mid-run to check "was this tool call already done"). This asymmetry is exactly why checkpoint/recovery is harder than tracing — tracing is fire-and-forget, checkpointing is a source of truth the loop depends on to make correctness decisions.
3. **The dashboard flow is entirely read-only and downstream:** RSCs query `runs`/`spans`/`eval_results`/`checkpoints` directly; no write path exists from the dashboard back into these tables (there is no "edit a trace" feature in scope).

## Checkpoint/Recovery — Concrete Design (Question 5, in depth)

### Is the trace store the same store as the resumable state, or different?

**Different logical models, decide store-sharing as an implementation detail, not a design merge.** The trace (`runs` + `spans`) answers "what happened, for observability" — it's an append-only, human-readable log optimized for rendering a waterfall. The checkpoint answers "what do I need to restart the computation" — it's a **mutable, overwritten-in-place, minimal-and-complete snapshot** optimized for fast, unambiguous reconstruction of agent state. Concretely:

- A span can be redundant, verbose, and lossy (buffered writes are fine) because it's for humans.
- A checkpoint must be exactly sufficient for the loop to resume as if it had never stopped, and every write must land before the next side-effecting action proceeds.

These are different access patterns (append vs. upsert) and different durability requirements (Pattern 3), so they should be **different tables**, and it's fine — even good, for cross-referencing — for them to live in the **same SQLite file/connection**. Do not try to derive the checkpoint from replaying the span log at resume time: spans capture "a tool call happened," but the checkpoint needs to capture "the tool call's result, keyed for idempotent lookup, plus the exact message array to feed back to the model" — deriving the latter from the former on every resume is solvable but adds a fragile reconstruction step for no benefit over just storing it directly. Store both; cross-link via `run_id`.

### What must be in a checkpoint

```sql
CREATE TABLE checkpoints (
  run_id           TEXT NOT NULL REFERENCES runs(id),
  step_index       INTEGER NOT NULL,
  messages         TEXT NOT NULL,     -- JSON: full Anthropic messages[] array as of this step
  pending_tool_call TEXT,              -- JSON: {id, name, args} if a tool was requested but
                                        -- not yet confirmed complete when this checkpoint was written
  status           TEXT NOT NULL,      -- 'step_started' | 'model_responded' | 'tool_requested' |
                                        -- 'tool_completed' | 'step_completed'
  written_at       INTEGER NOT NULL,
  PRIMARY KEY (run_id, step_index)
);

CREATE TABLE tool_results (
  run_id           TEXT NOT NULL,
  step_index       INTEGER NOT NULL,
  tool_call_id     TEXT NOT NULL,       -- Anthropic's tool_use id — natural idempotency key
  idempotency_key  TEXT NOT NULL,       -- derived: hash(tool_name + args) as a fallback/second check
  result           TEXT,                 -- JSON result, NULL if not yet completed
  completed_at     INTEGER,
  PRIMARY KEY (run_id, tool_call_id)
);
```

Minimum required fields, and why each one is load-bearing:
- **`messages[]` (full array, not a diff)** — the model needs the complete conversational context to continue; storing incremental diffs is a premature optimization that adds reconstruction risk for a project at this scale (dozens of steps, not thousands).
- **`step_index`** — the resume point; also what lets recovery tell "was step N ever finished" without needing to parse `messages[]`.
- **`tool_call_id` + result, in a separate table keyed for idempotent lookup** — this is the actual hard part (see below).
- **`status` enum on the checkpoint row** — distinguishes "the model responded but the tool hasn't run yet" from "the tool ran but we crashed before recording the model saw the result," which are different resume behaviors.

Checkpoint writes happen **before** the risky action they guard, not after — this is the "write-ahead" principle: write `status='tool_requested'` with the tool call's id/args *before* invoking the tool, then write `status='tool_completed'` with the result *after*. If the process dies between these two writes, recovery finds a `tool_requested` row with no matching `tool_results` entry and knows unambiguously that this specific tool call's completion status is unknown — which is precisely the scenario idempotency has to handle, not a corner case to hand-wave.

### Idempotency of tool replay — the actual hard problem

The project's four tools (`read_file`, `run_linter`, `run_tests`, `search_codebase`) are a favorable case: **all four are naturally idempotent** — none of them mutates external state (no "send email," no "book flight," no git commit). Re-running any of them on resume produces the same result (modulo the underlying files changing between runs, which won't happen inside a single eval fixture repo). This is worth stating explicitly in the design, because it means Tracewell can adopt the simplest possible recovery policy and still be correct:

**Recovery policy: "re-execute any tool call that isn't confirmed complete."** On resume, for the checkpoint's `pending_tool_call` (if any), check `tool_results` for a row with that `tool_call_id`. If found, reuse the stored result and never re-invoke the tool (this handles the case where the tool actually ran and only the *next* checkpoint write was lost). If not found, re-invoke the tool. Because all tools here are read-only/side-effect-free, re-invocation is always safe — no general-purpose idempotency-key deduplication against a live external system is needed.

**Why still model idempotency keys even though it's not strictly needed here:** this is precisely the point the teardown document should make explicit, since it's the most interesting general lesson in the project: idempotency is a property of the *tool*, not the *recovery system* — the recovery system's job is only to correctly determine "did this side effect happen," and it can only give the tool author a `tool_call_id` to key on. Modeling the `tool_results` table with an idempotency key (even though this project's tools don't need dedup logic on the reuse side) is what makes the checkpoint design honestly generalize to a version of this project with a mutating tool (e.g., "open a PR," "post a comment") — worth one paragraph in the teardown, and worth keeping the schema shape even though the current tools don't exercise the hard case.

### Recovery is a distinct process/entrypoint, not a hidden retry inside the loop

Resume should be a first-class CLI path — `agent resume <run_id>` — that: (1) loads the latest checkpoint row for `run_id`, (2) reconstructs `messages[]` and pending state, (3) re-enters `agent/loop.ts` at `step_index + 1` (the loop itself doesn't know it's resuming vs. starting fresh — this is what keeps the agent loop free of recovery-awareness, mirroring the tracing boundary principle). The **resumable-loop wrapper** (`checkpoint/resumable-loop.ts`) is the seam, analogous to `tracing/instrument.ts` — it wraps the same unmodified `agent/loop.ts`.

### Trace continuity across a crash/resume

When a resumed run continues, new spans should carry the **same `run_id`** as the original crashed run (not a new trace) — the dashboard's trace waterfall for that run then shows a visible time gap between the last pre-crash span and the first post-resume span, which is itself a useful, demoable artifact ("here's the trace showing the crash and resume boundary"). Mark the `runs.status` transition explicitly: `running → crashed → resuming → completed`, so the dashboard can render a resume badge.

## Scaling Considerations

This project is explicitly out-of-scope for scale (no multi-tenancy, no high-throughput trace store), so this section is about "what breaks first at the sizes this project will actually hit" rather than production scaling advice.

| Scale | Architecture Adjustments |
|-------|---------------------------|
| Dev / single demo run (this project's actual scale: dozens of runs, hundreds of spans) | SQLite + `better-sqlite3`, synchronous checkpoint writes, buffered span writes — no changes needed |
| A larger eval sweep (100s of cases, run repeatedly) | Batch the eval task-runner with bounded concurrency (e.g., 3–5 concurrent cases against the Claude API to respect rate limits); SQLite handles this fine under WAL mode since eval writes are still low-volume relative to what WAL was designed for |
| Dashboard phase, Supabase migration | Move `runs`/`spans`/`eval_results`/`checkpoints` schemas to Postgres with the same column shapes; swap `db/sqlite.ts` for `db/supabase.ts` behind the same query functions; JSON columns (`attributes`, `messages`) map directly to Postgres `jsonb` |

### Scaling Priorities

1. **First (and only realistic) bottleneck:** Claude API rate limits during eval sweeps, not the local database. Mitigate with a small concurrency limiter (e.g., `p-limit`) in the eval task runner — this is a two-line fix, not an architecture change.
2. **Second, theoretical:** SQLite write contention if the dashboard were reading live while an eval sweep writes heavily — WAL mode already handles concurrent readers/single-writer fine at this scale; not worth engineering further given the project's explicit "no high-performance trace store" scope decision.

## Anti-Patterns

### Anti-Pattern 1: Letting the agent loop import from `tracing/` or `checkpoint/`

**What people do:** Add a `span.setAttribute(...)` call, or a `saveCheckpoint(...)` call, directly inside the agent's decision loop "just this once" because it's convenient at the exact point where the interesting data is available.
**Why it's wrong:** It defeats the stated architectural goal (instrumentation boundary) and — concretely — makes it impossible to demonstrate the "no framework, agent stays pure" story the project's teardown depends on. It also means testing the agent loop in isolation now requires mocking the tracing/checkpoint subsystems.
**Do this instead:** Every piece of data tracing or checkpointing needs must be observable from *outside* the loop at the two seams (model call, tool call) or at the loop's own iteration boundary (agent-turn). If some data seems only available "from inside," that's a signal the seam needs to return that data (e.g., have `tool.run()` return `{ result, metadata }` rather than logging metadata internally), not that tracing should reach in.

### Anti-Pattern 2: Merging the checkpoint table into the spans table ("just add a `resumable` flag to spans")

**What people do:** Since spans already capture tool calls and their results, it's tempting to add a `checkpoint_data` column to `spans` and call the latest tool-call span "the checkpoint."
**Why it's wrong:** Spans are buffered (durability trade-off deliberately accepted for observability) — using them as the recovery source of truth silently reintroduces the exact data-loss risk Phase 3 exists to eliminate. It also conflates two different write disciplines (append-only vs. upsert-with-guaranteed-ordering) into one table, which will produce confusing migration code at the Supabase step.
**Do this instead:** Keep `checkpoints`/`tool_results` as separate, synchronously-written tables from day one of Phase 3, even though it's more schema to write. Cross-reference by `run_id` for the dashboard, don't merge storage.

### Anti-Pattern 3: Building a full OpenTelemetry pipeline (SDK + Collector + exporters) for a one-process app

**What people do:** Reach for `@opentelemetry/sdk-node`, `@opentelemetry/exporter-trace-otlp-http`, and a local Collector because "that's how real tracing works," then spend a disproportionate amount of the project's time budget on YAML collector config and exporter plumbing.
**Why it's wrong:** OTel's value is vendor-neutral interop across many services/languages; a single-process, single-language, single-store project gets none of that value and pays real complexity tax (protobuf schemas, context propagation setup, collector deployment) for it. It also obscures the exact mechanism (AsyncLocalStorage, span tree, SQLite write) that this project exists to teach the author.
**Do this instead:** Build the bespoke span model directly (Pattern 1), borrowing OTel's *attribute naming* for credibility, not its runtime. If a future version of the project wants OTel-compatibility for export to a real backend, that's a clean, additive Phase 5+ — not a Phase 1 requirement.

### Anti-Pattern 4: Treating "eval mode" as a different code path from "live agent mode"

**What people do:** Write a separate, simplified invocation path for eval cases (no tracing overhead, no checkpointing) to make the eval harness faster/simpler to build.
**Why it's wrong:** Directly undermines the stated demo ("bug → trace → eval catches regression") — if the eval path doesn't produce real traces through the real instrumented agent, there's no trace to show explaining *why* the eval caught the regression.
**Do this instead:** The eval task runner calls the exact same instrumented `agent/loop.ts` entrypoint a live CLI invocation would use (Pattern 4). Any eval-specific behavior (e.g., injecting a fixed fixture repo path) is a parameter, not a fork.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|----------------------|-------|
| Claude API (agent + LLM-judge scorer) | Direct SDK client (`@anthropic-ai/sdk`), wrapped once via `instrumentModelCall()` at construction time | Both the agent's own model calls and the eval harness's judge calls should go through the same instrumented client so judge calls also produce spans (useful for debugging "why did the judge score this low") |
| Supabase (dashboard phase only) | Postgres client behind the same `db/` query interface used for SQLite | Migrate schema with a one-time script; keep JSON/JSONB parity; do not introduce Supabase-specific features (RLS, realtime) since there's no multi-tenancy requirement |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|----------------|-------|
| `agent/` ↔ `tracing/` | One-directional HOF wrapping at two seams (model call, tool call) + one explicit loop-boundary wrap (agent-turn) | `agent/` has zero imports from `tracing/`; `tracing/` imports the Claude client type only for wrapping |
| `agent/` ↔ `checkpoint/` | `checkpoint/resumable-loop.ts` wraps `agent/loop.ts` from outside, writing before/after each step | Same boundary discipline as tracing; the loop itself is checkpoint-unaware |
| `tracing/` ↔ `checkpoint/` | Share a `run_id`/`trace_id` namespace and (likely) a physical SQLite connection, but separate tables and separate flush/durability policies | Do not let one module import the other's write path; cross-reference only by id at read time (dashboard, recovery) |
| `eval/` ↔ `agent/` + `tracing/` | Calls the same instrumented entrypoint as live use; reads back the trace via `run_id` after the run completes, to pass to scorers | No special "eval trace" schema — `runs.kind = 'eval_case'` is the only distinction |
| `db/` ↔ everything else | All modules query through `db/sqlite.ts` (or `db/supabase.ts` post-migration) functions, never raw SQL scattered across modules | This is what makes the SQLite→Supabase swap a one-file change |
| `dashboard/` (Next.js) ↔ `db/` | Server Components import `db/` directly and query at render time; no API route needed for the initial page loads (trace list, trace detail, run comparison, cost breakdown) since these are all server-rendered on request | Add an API route only if/when a feature needs client-side interactivity beyond what a form-action/server-action can do (e.g., live-polling a running eval sweep) — not needed for the stated MVP dashboard scope |

## Suggested Build Order (with explicit dependencies)

The project's stated phase order is: **agent → tracing → eval → checkpoint/recovery → quality gate → dashboard.** Validating this against the actual dependency graph:

```
agent/  (no deps)
   │
   ▼
tracing/  (depends on: agent's two seams existing to wrap)
   │
   ├──────────────┐
   ▼              ▼
eval/          checkpoint/     (both depend on: agent/ + tracing/, but NOT on each other)
   │              │
   ▼              │
gate/  (depends on: eval/ only)
   │              │
   └──────┬───────┘
          ▼
 dashboard/  (depends on: whatever tables exist by the time it's built — runs, spans,
              eval_results, checkpoints — reads all of them)
```

**This validates the author's stated order, with one clarification and one caveat:**

1. **agent → tracing is a hard dependency and correctly first/second.** You cannot wrap seams that don't exist yet. Confirmed correct.
2. **tracing → eval is a real dependency, but softer than it looks.** Eval *could* technically be built against an untraced agent (scorers only need the output, not the trace) — but Pattern 4 above shows that doing so would forfeit the "eval case IS a trace" relationship the project explicitly wants for its demo. So: tracing before eval is the right call not because eval can't function without it, but because building eval first would create a second, divergent invocation path that later has to be reconciled with tracing. Confirmed correct, and worth stating why in the roadmap.
3. **eval → checkpoint/recovery ordering: the dependency graph does NOT actually require eval before checkpoint/recovery.** Checkpoint/recovery depends only on `agent/` + `tracing/` (it needs spans/runs concepts and the agent's step structure), not on the eval harness. The stated order (eval, then checkpoint/recovery) is a legitimate and reasonable *sequencing choice* — eval is lower-risk and delivers a demoable artifact (comparison table) faster, which fits the "short, interruptible sessions" constraint — but it is not dependency-mandated. **If the author instead needed the eval harness to validate correctness of *resumed* runs** (e.g., an eval case that specifically tests "agent crashes mid-case, resumes, produces the same score as an uninterrupted run") — a genuinely strong demo idea worth considering — then checkpoint/recovery would need to exist *before* that specific eval case could be written, though not before the eval harness itself. Recommendation: keep the stated order (it's fine), but flag this as an opportunity — a "resume produces an equivalent trace/score" eval case is a high-value addition to the eval dataset once checkpoint/recovery exists, so don't design the eval dataset schema in a way that forecloses adding it later (e.g., allow a case to specify "inject a crash at step N").
4. **checkpoint/recovery → quality gate: correctly ordered relative to eval, but the gate itself only depends on eval, not on checkpoint/recovery.** The gate (threshold check on eval scores) has no functional dependency on checkpoint/recovery at all. The stated grouping "Phase 3: checkpoint/recovery + quality gate" bundles two independent-dependency features into one phase; the project's own Key Decisions table already reflects a later split of these into separate phases ("Split checkpoint/recovery and the eval quality gate into separate phases... finer slices give clearer wins and fit part-time sittings") — this is well-supported by the dependency graph, since the gate could as easily be built as a small addendum to the eval phase (it's a ~30-line comparator over eval output) rather than bundled with the much harder checkpoint/recovery work. **Recommendation: build the quality gate as a light final step of the eval phase (or its own very short phase immediately after eval), and let checkpoint/recovery stand alone as its own phase** — this matches the project's already-stated intent to split them and is dependency-consistent.
5. **dashboard last is correct and dependency-mandated** — it's the only component that reads from every other store (`runs`, `spans`, `eval_results`, `checkpoints`), so it necessarily comes after all of them have a real schema to read. It's also correctly where the SQLite→Supabase migration is scoped, since it's the first component for which a hosted, multi-page, network-accessible read path actually matters (a CLI eval runner has no need for Postgres).

**Revised recommended order** (tightening the original without contradicting it):

1. `agent/` — pure loop, no deps
2. `tracing/` — wraps agent; establishes `runs`/`spans` schema and AsyncLocalStorage context
3. `eval/` — dataset + task runner (reuses instrumented agent) + scorers; produces `eval_results`; **build the quality-gate threshold check here as a final small step**, not as a separate later phase, since it has no independent dependency
4. `checkpoint/recovery` — stands alone as the hardest phase; depends only on `agent/` + `tracing/`, so it could technically move earlier, but sequencing it after eval is reasonable for morale/risk reasons (a working demoable eval harness exists before tackling the hardest problem) — optionally add one eval case exercising "crash and resume produce an equivalent score" once this phase lands, to make the two phases visibly reinforce each other in the demo
5. `dashboard/` — reads everything; this is also where SQLite → Supabase migration happens, and where the prepared demo case (bug → trace → eval regression) gets its visual presentation

## Sources

- [OpenTelemetry GenAI Observability blog](https://opentelemetry.io/blog/2026/genai-observability/) — HIGH confidence, official OTel source, on GenAI span kinds and attribute naming (`gen_ai.usage.input_tokens`, `chat`/`invoke_agent`/`execute_tool` span types)
- [Langfuse Observability Data Model docs](https://langfuse.com/docs/observability/data-model) — HIGH confidence, official Langfuse docs, on trace/observation/parent_observation_id structure and SPAN/GENERATION/EVENT typing
- [Langfuse Trace IDs & Distributed Tracing docs](https://langfuse.com/docs/observability/features/trace-ids-and-distributed-tracing) — HIGH confidence, official docs
- [Braintrust Span reference (Node.js SDK)](https://www.braintrust.dev/docs/reference/libs/nodejs/interfaces/Span) — HIGH confidence, official Braintrust reference, on `span_id`/`root_span_id`/`span_parents`/`span_attributes.type`
- [Braintrust Advanced tracing patterns](https://www.braintrust.dev/docs/instrument/advanced-tracing) — HIGH confidence, official docs
- [Node.js Asynchronous context tracking docs](https://nodejs.org/api/async_context.html) — HIGH confidence, official Node.js docs, on AsyncLocalStorage propagation across await/Promise/setTimeout
- [Platformatic: The Hidden Cost of Async Context in Node.js](https://blog.platformatic.dev/the-hidden-cost-of-context) — MEDIUM confidence, vendor blog, on ~7% AsyncLocalStorage overhead and stale-context pitfalls
- [better-sqlite3 performance docs (WiseLibs)](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/performance.md) — HIGH confidence, official library docs, on WAL mode default synchronous=NORMAL trade-offs
- [SQLite commits are not durable under default settings](https://avi.im/blag/2025/sqlite-fsync/) — MEDIUM confidence, independent technical blog, cross-checked against official SQLite forum discussion on synchronous pragma durability semantics
- [LangGraph Persistence / Checkpointers docs](https://docs.langchain.com/oss/python/langgraph/checkpointers) and [LangGraph JS persistence guide](https://langgraphjs.guide/persistence/) — HIGH/MEDIUM confidence, official + community, on checkpoint-per-superstep, thread_id resumption, SqliteSaver/PostgresSaver backends — used as the closest existing analog for Tracewell's checkpoint design, not copied directly
- [Idempotent AI Agents: Retry-Safe Patterns for Production](https://www.buildmvpfast.com/blog/idempotent-ai-agent-retry-safe-patterns-production-workflow-2026) and [Durable Execution for AI Agent Runtimes (Zylos Research)](https://zylos.ai/research/2026-04-24-durable-execution-agent-runtimes/) — MEDIUM confidence, industry commentary, cross-checked against each other, on write-ahead checkpointing and "redelivery means the tool must tolerate a second invocation" — directly informs the idempotency-key design in this document
- [Next.js App Router: Fetching Data (official docs)](https://nextjs.org/learn/dashboard-app/fetching-data) — HIGH confidence, official Next.js docs, on Server Components querying databases directly without an API layer
- Project files `.planning/PROJECT.md` and `planning/PLAN.md` — primary source for constraints, stated phase order, and out-of-scope boundaries

---
*Architecture research for: TypeScript agent tracing + eval + checkpoint/recovery system (Tracewell)*
*Researched: 2026-09-08*
