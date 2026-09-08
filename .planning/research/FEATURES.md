# Feature Research

**Domain:** Agent observability + evaluation platforms (Braintrust, LangSmith, Langfuse, Arize Phoenix, W&B Weave, Laminar)
**Researched:** 2026-09-08
**Confidence:** MEDIUM (high agreement on core vocabulary/architecture across vendors; lower confidence on any single vendor's exact UI details, since much of the source material is vendor blog/comparison content with self-interest bias — treated as MEDIUM even where a vendor's own docs are cited, and explicitly flagged LOW where a claim comes from a single non-official source)

## Context: what this category actually is

Every platform in this space (Braintrust, LangSmith, Langfuse, Arize Phoenix, W&B Weave, Laminar) converges on the same core vocabulary, independent of vendor:

- **Trace** = one end-to-end execution (a request, a conversation turn, an agent run), made of **spans** in a parent-child tree.
- **Span** = one unit of work (an LLM call, a tool call, a retrieval step) with inputs, outputs, timing, and (for LLM spans) token usage.
- **Dataset** = a versioned set of input/expected-output examples.
- **Experiment / run** = one evaluation pass of a dataset through a scorer pipeline, producing per-case and aggregate scores.
- **Scorer** = code (hard assertion) or LLM-as-judge, sometimes human annotation.

This convergence is a strong signal: it is the load-bearing abstraction of the category, not incidental UI. Tracewell should adopt the same vocabulary (trace/span/dataset/experiment/scorer) rather than invent new terms — this is free credibility in an interview setting and it is what the fixture repo, eval harness, and dashboard all hang off of.

The one place the category is **not** unanimous is checkpoint/recovery — see the dedicated section below, since it directly addresses the author's stated belief.

## Feature Landscape

### Table Stakes (must exist or the demo does not land)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Trace capture: spans + parent-child nesting | Every platform in the category organizes execution as a nested span tree (Braintrust, LangSmith, Langfuse, Phoenix, Weave, Laminar all use this model identically). Without nesting there is no "trace," just a flat log. | MEDIUM | The hard part is schema design, not instrumentation — a manual push/pop span stack around the Phase 0 agent loop is a day's work once the schema is right. Already an Active requirement. |
| Span waterfall / tree view in trace UI | This is the primary artifact every platform's "trace detail" screen is built around (Braintrust's Spans/Thread/Timeline views, LangSmith's trace tree, Langfuse's nested view). It is the thing an interviewer will actually look at. | MEDIUM | Recursive tree render + horizontal timeline bars. No virtualization needed at demo scale (dozens of spans, not thousands). |
| Tool call inputs/outputs shown per span | Braintrust: "Messages section shows input and output messages, tool calls, and annotations." This is what makes a trace explain a bug, not just time it. | LOW | Direct dependency: trace schema must store raw args/results per tool-call span, not just a summary string. |
| Token + cost per span, rolled up to parent | Universal across Braintrust, LangSmith, and Weave: cost/tokens attached to each LLM span and propagated up the tree ("cost propagated from child spans to parent spans" — Braintrust; LangSmith aggregates cost at trace and per-run level). | LOW-MEDIUM | **Dependency:** requires the trace schema to persist input/output token counts per LLM call (from the Claude API `usage` field) *and* a static price-per-model table, since the API does not return a dollar cost directly. Already an Active requirement (cost/token breakdown). |
| Latency shown per span, with breakdown | Every platform shows duration per span and a timeline of where time went (Braintrust's "Timeline" view explicitly framed as "execution flow and token efficiency"). | LOW | Just start/end timestamps per span; "breakdown" = summing by span type (model call vs. tool call) for a demo-friendly stat, trivial once spans have timestamps. |
| Error surfacing on the failing span | Braintrust: "span errors appearing alongside the input and output that produced them." This is table stakes because a trace's whole job is explaining *why* something broke. | LOW-MEDIUM | **Gap in stated scope — see below.** Not explicitly named in the Active trace-schema requirement, but the demo's core value prop ("bug → trace explains why") is unachievable without it. |
| Datasets of eval cases | Universal (Braintrust datasets, Langfuse datasets, Phoenix datasets, Weave datasets). | LOW | Already decided: fixture repo of 10-20 seeded bugs. |
| Scorers: hard assertions + LLM-as-judge | Universal. Langfuse and Weave both explicitly support combining deterministic/code scorers with LLM-judge scorers in the same run; Weave literally supports "three scorer types (code, LLM judge, human) combine in one experiment." | LOW (hard assertions) / MEDIUM (LLM-as-judge) | Already decided. LLM-judge cost: prompt design + reliable JSON-mode parsing + calibration against the fixture repo's known-answer cases. |
| Experiments/runs as a first-class stored entity | Every platform stores each eval pass as a named, comparable run (Braintrust experiments, Langfuse dataset runs, Phoenix experiments, Weave evaluations). | LOW-MEDIUM | Needed before comparison is possible — see dependency graph. |
| Run-over-run comparison / diff view | Braintrust: "enable diff mode... each test case expands to show a sub-row per experiment... compare outputs, scores, and metadata inline," color-coded regressions/improvements. This is close to universal and is explicitly required by the project. | MEDIUM | Already decided (dashboard shows run comparison diff). Needs a designated "baseline" run — see gap below. |
| Score aggregation (per-run overall score) | Implicit prerequisite for both comparison and the quality gate — every platform rolls per-case scores into a run-level number. | LOW-MEDIUM | **Gap in stated scope — see below.** Not named explicitly as a requirement, but the quality gate and run comparison both depend on it existing. Needs a defined formula for combining hard-assertion pass-rate with LLM-judge score into one number (or a small vector of per-scorer aggregates) — a design decision, not just code. |
| CI quality gate on score threshold | Braintrust's `eval-action` runs on every PR, posts a score summary comment, and blocks merge below threshold. This is a well-established pattern (also see Openlayer's "CI/CD Evaluation Gates" framing) and is an Active requirement. | LOW-MEDIUM | Depends on score aggregation existing first. Already scoped as "warns" (not hard-block) per the Active requirements — reasonable simplification for a solo demo. |

### Differentiators (what makes this worth showing an interviewer)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Checkpoint/recovery (durable execution across a tool-calling loop) | **This is a genuine differentiator, not table stakes — confirmed by research.** See dedicated section below for the full argument. In short: mainstream eval/observability platforms explicitly decline to own this ("Braintrust cannot suspend an active execution, hold a tool call for approval, or resume from a durable runtime checkpoint... an intended boundary between an evaluation platform and an execution runtime"). Building it anyway, scoped to one hand-written agent, is exactly the kind of "infra layer" understanding that separates a portfolio piece from a CRUD dashboard. | HIGH | Already flagged in PROJECT.md as "the hardest and most interesting work." The code-review agent's tools (`read_file`, `run_linter`, `run_tests`, `search_codebase`) are all read-only/idempotent — no side effects to worry about on replay (no "did this email get sent twice" problem). This materially lowers the complexity versus the general case and is worth stating explicitly in the teardown as *why* this agent was a good choice for demonstrating durable execution safely. |
| Trace → eval-case conversion | Braintrust: "when a user reports a bad response, teams can convert the trace directly into an evaluation case." Turns the demo into a closed loop: a bug is found in a live trace, promoted into the fixture repo, and the regression test now exists. | LOW-MEDIUM (if the fixture format and trace schema are designed to interoperate from the start) | Optional stretch — strengthens the "bug → trace → eval" narrative arc but is not required for it to work; the fixture repo already provides deterministic ground truth. Flag as a stretch goal for Phase 4 if time allows. |
| Cost/token breakdown *in the same view as* the trace waterfall and the run comparison | Table stakes individually, but wiring all three into one coherent dashboard (rather than three disconnected reports, which is how some smaller tools ship it) is what makes the demo feel like a real product rather than a script's console output. | MEDIUM | This is really "table stakes features, integrated well" — the differentiation is UX cohesion, not a new capability. Counted here because it's what an interviewer actually sees. |
| An explicit, defensible design rationale for what was cut | None of the reference platforms narrate *why* they don't do X — they just don't do it. Tracewell's teardown can. A one-page "why no multi-tenancy, why no columnar store, why checkpoint/recovery instead of a trend dashboard" argument is a differentiator that costs zero implementation time and directly serves the project's actual audience (an interviewer evaluating judgment, not a user evaluating a product). | LOW (writing, not code) | This is the "architecture teardown" deliverable already in scope — make sure it explicitly contrasts against Braintrust/LangSmith/Langfuse feature-for-feature, not just describes what was built. |

### Anti-Features (deliberately NOT built)

| Feature | Why Requested (in the category) | Why Problematic (for this project) | Alternative |
|---------|---------------------|------------------|-------------|
| Multi-tenancy / RBAC / SSO | Required by every commercial platform because they serve many customer orgs. | No external users; solo demo. Pure overhead with zero payoff. | Single implicit user/workspace. Already Out of Scope in PROJECT.md. |
| Sampling at scale (head/tail sampling, trace volume budgets) | Needed once trace volume hits production scale (millions/day) — this is *why* Brainstore-class columnar stores exist. | Demo volume is tens to low hundreds of traces total. Sampling logic is pure complexity with no scale to justify it. | Capture 100% of traces, always. |
| High-performance / columnar trace store (Brainstore-class) | Needed to query and aggregate over millions of trace rows cheaply. | Already Out of Scope in PROJECT.md ("the learning is in the schema, not the storage engine"). SQLite/Postgres handles this project's scale trivially. | SQLite (Phases 1-3) → Supabase/Postgres (dashboard phase), per the existing Key Decision. |
| Prompt playground (live in-browser prompt iteration UI) | Phoenix, Datadog, and Agenta all ship this — useful when prompt engineers iterate without touching code. | The agent's prompts are edited in TypeScript source as part of the hand-built loop; a separate browser-based prompt editor duplicates that surface and adds a whole product (prompt versioning + live model calls from the UI) with no bearing on the tracing/eval/checkpoint learning goals. | Edit prompts in code, re-run the CLI eval harness. |
| Annotation queues / structured human review workflow | LangSmith, Datadog, and Agenta all build this for teams triaging live production traffic that needs human labeling before it becomes a test case. | Already decided: ground truth comes from a deterministic fixture repo of seeded bugs, not human-labeled production traffic. There is no reviewer, no queue, no labeling backlog. | Fixture repo with pre-known expected outcomes (already the Key Decision). |
| Alerting / paging integrations (Slack, PagerDuty on regression) | Needed once evals run unattended in production monitoring pipelines. | The author is the only consumer of results and drives the CI gate manually or via a demo run. No unattended monitoring exists to page about. | Console/CI output + PR comment equivalent (or just terminal output) from the quality-gate command. |
| Score-over-time trend dashboard | Every platform eventually adds this once customers have enough historical runs to plot. | Already explicitly deferred to v2 in PROJECT.md — a trend needs many runs to say anything, and this project will produce few. | Run-over-run diff (two named runs) is the substitute for now. |
| Full prompt/agent semantic versioning model | Braintrust/LangSmith build rich version graphs (prompt versions, deployment history, rollback) because customers manage many prompts across many environments over months. | Already explicitly scoped down in PROJECT.md — "comparison built only as deep as the demo needs." A full versioning model is Braintrust's product surface, not the learning objective. | Git branches per phase (already a Key Decision) stand in for version history; comparison operates on two named runs, not a version graph. |
| Statistical significance testing on score diffs | Some platforms (not even Braintrust, per this research — "Braintrust's evaluation results page documents no statistical confidence or significance concepts") flag this as a gap worth filling. | At n=10-20 fixture cases, significance testing is mostly theater — the sample is too small for it to mean anything, and it's genuinely hard to get right (multiple comparisons, non-normal score distributions). | Show raw score deltas per case, color-coded improve/regress — same simplification the market leader itself ships. |
| Generic OpenTelemetry-native, multi-framework instrumentation SDK | Phoenix and Laminar both build this because they support arbitrary customer stacks (LangChain, CrewAI, LlamaIndex, Vercel AI SDK, etc.) with "1 line of code" auto-instrumentation. | This project has exactly one hand-written agent with no framework by design — building a generic, framework-agnostic SDK is solving a problem (support N frameworks) that does not exist here and would consume the "understand the tool-calling loop at a primitive level" learning budget on SDK plumbing instead. | Hand-wire span push/pop calls directly into the one agent loop. |
| Production-issue auto-clustering / "data flywheel" (LangSmith Engine-style) | Advanced ML feature that clusters live failures into prioritized issues and auto-generates regression tests. | Sophisticated, ML-driven, built by a well-funded team over years; wildly out of proportion to a solo 4-8 week project and orthogonal to the checkpoint/recovery and eval-gate learning goals. | Manual trace → eval-case promotion (see Differentiators) if pursued at all. |

## Checkpoint/Recovery: is it table stakes or a differentiator? (dedicated finding)

The author's instinct — that checkpoint/recovery is core, and possibly a differentiator rather than table stakes — is **confirmed by research**, with an important nuance:

- **Braintrust, Langfuse, Arize Phoenix, and W&B Weave do not offer checkpoint/recovery or durable execution.** They are eval/observability layers that record what happened; they do not own runtime state or execution control. The clearest evidence: Braintrust explicitly states it "cannot suspend an active execution, hold a tool call for approval, or resume from a durable runtime checkpoint," framing this as "an intended boundary between an evaluation platform and an execution runtime" — i.e., a deliberate, stated non-goal, not a missing feature they're racing to add.
- **Checkpoint/resume-style capability does exist in this ecosystem, but it lives one layer down**, in agent *orchestration frameworks* (LangGraph's checkpointer/state-rewind, exposed via LangGraph Studio inside the LangSmith umbrella) or in general-purpose *durable execution engines* (Temporal, Inngest, Restate) that are not observability products at all — they are workflow runtimes that happen to be popular for wrapping LLM agents.
- **Conclusion for Tracewell:** checkpoint/recovery is not table stakes for an "observability + eval platform" in the reference category — no mainstream eval/observability product ships it as a peer feature to tracing and evals. That makes it a genuine **differentiator** for this project specifically: Tracewell isn't cloning a gap that exists in Braintrust by accident, it's deliberately fusing a capability that the market keeps in a separate product category (durable execution runtimes) into the same tool that does tracing and evals. That fusion — and the reasoning for why it belongs together at this project's scale — is a strong, specific thing to defend in an interview, precisely because it's not something Braintrust/Langfuse/Phoenix do.
- **Complexity reality check:** this is legitimately the hardest phase (already flagged as such in PROJECT.md). The durable-execution research is consistent on the two hard problems: (1) LLM output is non-deterministic and must be *recorded once, replayed from the record* on resume, never re-called; (2) any tool call with side effects needs an idempotency key to avoid duplicate effects on replay. Tracewell gets a genuine simplification here that's worth calling out explicitly in the teardown: all four of the code-review agent's tools (`read_file`, `run_linter`, `run_tests`, `search_codebase`) are read-only and idempotent by nature — re-running `run_tests` twice on resume produces the same observable result and no duplicate side effect. This removes the idempotency-key problem that makes durable execution hard for agents with side-effecting tools (e.g., "send an email," "charge a card"), leaving primarily the "record LLM output once, don't re-call the model on replay" problem to solve. That's still real work, but meaningfully smaller than the general case — and is worth stating as a deliberate reason this agent was chosen.

## Feature Dependencies

```
Trace schema (spans + parent-child + timing)
    └──requires──> Trace capture wrapper around agent loop (Phase 0 agent must exist first)

Trace schema + LLM usage field
    └──requires──> Cost/token breakdown per span
                       └──requires──> Static per-model price table (not returned by the Claude API)
                       └──enhances──> Span waterfall view (inline cost/token badges)

Trace schema + explicit span "status"/error field
    └──requires──> Error surfacing in trace UI
                       └──enables──> "trace explains why" demo narrative (core value prop)

Dataset (fixture repo of seeded bugs)
    └──requires──> Eval harness (runs agent against each case)
                       └──requires──> Scorers (hard assertions + LLM-as-judge)
                                          └──requires──> Per-case scores
                                                             └──requires──> Score aggregation formula (per-run score)
                                                                                └──requires──> Quality gate (threshold check)
                                                                                └──requires──> Run comparison / diff view
                                                                                                   └──requires──> ≥2 named, stored experiment runs
                                                                                                   └──requires──> A designated "baseline" run

Eval harness runs
    └──enhances──> Trace store (each eval-run agent execution also produces a trace, viewable in the same waterfall UI)

Checkpoint/recovery
    └──requires──> Trace/state schema capable of persisting intermediate step state (likely extends the span schema rather than a separate store)
    └──requires──> Idempotent or read-only tool set (satisfied by this project's agent: read_file, run_linter, run_tests, search_codebase are all side-effect-free)
    └──requires──> Recorded LLM response (never re-called on resume/replay)
    └──enhances──> Eval harness reliability (a flaky mid-run failure during a 10-20 case eval sweep can resume instead of restarting the whole sweep)

Dashboard (Next.js)
    └──requires──> Trace store queryable (Phase 1)
    └──requires──> Experiment/run storage + score aggregation (Phase 2)
    └──requires──> Run comparison data model (two runs + per-case score deltas)
    └──conflicts with──> Score-over-time trend view (explicitly deferred to v2; do not build the query/aggregation plumbing for it now, it would be wasted work)

Session/thread grouping (grouping multiple traces under one conversation)
    └──not required by──> This project's usage pattern (one trace per agent run against one PR/file; no multi-turn conversation threading like a chatbot). Recommend skipping — see Gaps below.
```

### Dependency Notes

- **Cost/token breakdown requires a static price table:** the Claude API returns `usage` (input/output token counts) but not a dollar figure. Every reference platform (Braintrust, LangSmith, Weave) computes cost client-side from tokens × a maintained price-per-model table. This table needs to live somewhere versionable (a small JSON/TS constant is enough) and be updated if the demo switches Claude model versions.
- **Score aggregation must exist before both the quality gate and run comparison can work** — this is a hidden middle step that the stated requirements jump over (they name "eval harness," "comparison," and "quality gate" but not the aggregation logic that sits between per-case scores and either of those). It needs a real design decision: e.g., weighted average of hard-assertion pass rate and LLM-judge score, or a small vector of named metrics shown separately rather than collapsed to one number. Recommend deciding this explicitly during requirements, not leaving it implicit in code.
- **Run comparison needs a "baseline" concept, not just "two runs":** Braintrust's diff view designates one experiment as the reference point. Tracewell's "comparison of two named runs" (already an Active requirement) should explicitly pick which one is "before" — this is a two-line design decision, not extra scope, but worth stating so the dashboard UI has an unambiguous "regressed vs. improved" direction to color-code.
- **Checkpoint/recovery enhances rather than blocks the eval harness:** it is not a hard prerequisite for evals to work (evals can just be slow/restart-from-scratch without it), but its value is highest precisely in the eval-sweep context — a 10-20 case sweep that fails on case 14 due to a transient API error benefits the most from resuming instead of re-running cases 1-13.
- **Error surfacing enhances the trace waterfall but is scoped by the trace schema, not the UI:** the UI work is trivial (a red badge/row) once the schema carries a status/error field per span. The dependency that matters is upstream: the trace-capture wrapper must catch and record exceptions per span rather than letting them bubble and abort the whole trace unrecorded.

## Gaps in the Project's Stated Scope

The following are things the demo's stated core value ("bug → trace explains why → eval catches the regression") actually needs, but are not named as explicit requirements in PROJECT.md today:

1. **Error/exception capture in the trace schema.** The Active requirements list "spans, parent-child nesting, latency, and cost" for the trace schema but do not mention error/failure state. Without an explicit error field per span, a trace cannot visually explain *why* something broke — it can only show that something took a certain amount of time and returned some output. This is load-bearing for the stated core value and should be added to the trace schema requirement explicitly, not left implicit.
2. **Score aggregation formula (combining hard assertions + LLM-judge into a run-level score).** Named nowhere directly; both "regression detection" and "quality gate" silently depend on it. Recommend surfacing this as its own small requirement/decision during roadmap/requirements so it doesn't get invented ad hoc mid-implementation.
3. **A defined "baseline" run for comparison.** "Comparison of two named runs" is stated, but not which one anchors the diff. Trivial to decide, easy to forget, and it changes what "regression" means in the UI (regressed *relative to what*).
4. **Idempotency/replay semantics for checkpoint/recovery, specifically "never re-call the LLM on resume."** PROJECT.md names checkpoint/resume as a requirement and correctly calls it the hardest phase, but doesn't yet state the one rule that makes it tractable: on resume, prior LLM responses must be replayed from the persisted trace, not re-requested from the API. This should be made an explicit design constraint in that phase's spec, since getting it wrong silently produces a "checkpoint" system that just re-runs everything with different random outputs — not real recovery.
5. **Session/thread grouping is table-stakes in the reference category but is likely *not needed* here** — flagged as a gap in the other direction. Because the code-review agent produces one trace per run (not multi-turn chat sessions), building session/thread grouping (a table-stakes feature for LangSmith/Braintrust/Langfuse because their users are mostly chatbots) would be scope creep with no payoff for this demo. Worth an explicit "we deliberately did not build this, here's why" line in the teardown rather than silently omitting a category-standard feature.

## MVP Definition

### Launch With (v1) — matches the project's stated Active requirements, reordered by dependency

- [ ] Hand-written agent loop (Phase 0) — everything else observes this
- [ ] Trace schema: spans, parent-child nesting, latency, cost, **and per-span error/status** (add the error field explicitly per Gap #1)
- [ ] Automatic trace capture wrapper producing one queryable trace per run
- [ ] Fixture repo of 10-20 seeded bugs (deterministic ground truth)
- [ ] Eval harness: hard assertions + LLM-as-judge scorers
- [ ] Score aggregation formula (per-run score) — make this an explicit decision, not an afterthought (Gap #2)
- [ ] Run comparison of two named runs, with an explicit baseline (Gap #3)
- [ ] Checkpoint/resume for the agent loop, built around record-once/replay-on-resume for LLM calls (Gap #4) — leverages this agent's idempotent, side-effect-free tools
- [ ] Quality gate warning below a score threshold
- [ ] Dashboard: trace waterfall viewer, run comparison diff, cost/token breakdown
- [ ] Prepared demo case (bug → trace → eval catches regression) exercising the error-surfacing path end to end
- [ ] Architecture teardown document, written to explicitly contrast against Braintrust/LangSmith/Langfuse feature-for-feature (differentiator, zero implementation cost)

### Add After Validation (v1.x, only if time remains within the 4-8 week window)

- [ ] Trace → eval-case conversion (promote a bad live trace directly into the fixture repo) — strengthens the closed-loop narrative but isn't required for it to already work

### Future Consideration (v2+, explicitly deferred per PROJECT.md)

- [ ] Eval-score-over-time trend view — needs many runs to be meaningful; this project won't produce enough
- [ ] Full prompt/agent versioning model beyond two named runs
- [ ] Any of the anti-features listed above, if the project ever needed to scale past "solo demo"

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Trace schema + capture (incl. error field) | HIGH | MEDIUM | P1 |
| Span waterfall UI | HIGH | MEDIUM | P1 |
| Cost/token breakdown | HIGH | LOW-MEDIUM | P1 |
| Dataset + hard-assertion scorers | HIGH | LOW | P1 |
| LLM-as-judge scorer | HIGH | MEDIUM | P1 |
| Score aggregation formula | HIGH | LOW-MEDIUM | P1 |
| Run comparison / diff view | HIGH | MEDIUM | P1 |
| Quality gate (warn on threshold) | MEDIUM-HIGH | LOW-MEDIUM | P1 |
| Checkpoint/recovery | HIGH (differentiator) | HIGH | P1 (per project's own framing — "core," not optional) |
| Trace → eval-case promotion | MEDIUM | LOW-MEDIUM | P2 |
| Session/thread grouping | LOW (not needed for this usage pattern) | LOW | P3 / skip |
| Score-over-time trends | LOW (too few runs to be meaningful) | MEDIUM | P3 / deferred to v2 |
| Statistical significance testing | LOW | MEDIUM-HIGH | Skip |
| Multi-tenancy, RBAC, sampling, columnar store, prompt playground, annotation queues | NEGATIVE (pure overhead for this audience) | HIGH | Skip |

## Competitor Feature Analysis

| Feature | Braintrust | LangSmith | Langfuse | Tracewell's Approach |
|---------|--------------|--------------|----------|-----------------------|
| Trace view | Spans / Thread / Timeline, three interchangeable views | Trace tree with token/cost rollups per run | Nested span view | One good waterfall view; skip the multi-view switcher — one well-built view beats three shallow ones |
| Cost/token | Per-span, rolled up to parent, estimated via price table | Per-span and per-thread aggregation | Per-span | Per-span + total, using a small static Claude price table |
| Eval scorers | Code + LLM-as-judge, diff-mode comparison table | Datasets + evaluators, "data flywheel" from production | Templated LLM-judge scorers (Hallucination, Relevance, etc.) + code scorers | Hard assertions (lint/test pass-fail) + one calibrated LLM-judge scorer, scoped to the fixture repo — no template library needed |
| Run comparison | Diff mode, color-coded regressions, sortable by delta | Side-by-side run comparison | Dataset experiment run comparison view | Two named runs, one designated baseline, color-coded deltas — same pattern, smaller scope |
| CI quality gate | Native GitHub Action, blocks merge below threshold | Not a primary marketed feature | Not a primary marketed feature | Warn (not hard-block) below threshold — Braintrust's pattern, deliberately softened per project scope |
| Checkpoint/recovery | Explicitly out of scope ("intended boundary") | Available only via LangGraph (a separate framework), not LangSmith itself | Not offered | **Built in, scoped to one agent with idempotent tools** — the project's stated differentiator |
| Prompt playground / annotation queues / sampling / columnar store | All present in some competitor's product line | Present (LangGraph Studio, annotation queues) | Present (annotation workflows) | Deliberately not built — see Anti-Features |

## Sources

- [Braintrust — Examine traces](https://www.braintrust.dev/docs/observe/examine-traces)
- [Braintrust — How to read a trace](https://www.braintrust.dev/foundations/how-to-read-a-trace)
- [Braintrust — How to track LLM token usage](https://www.braintrust.dev/articles/how-to-track-llm-token-usage-2026)
- [Braintrust — Compare experiments](https://www.braintrust.dev/docs/evaluate/compare-experiments)
- [Braintrust — Quality gate definition](https://www.braintrust.dev/encyclopedia/quality-gate)
- [Braintrust — Best AI Eval Tools for CI/CD Pipelines](https://www.braintrust.dev/articles/best-ai-evals-tools-cicd-2025)
- [Braintrust — LangSmith vs. Braintrust](https://www.braintrust.dev/articles/langsmith-vs-braintrust)
- [Openlayer — CI/CD Evaluation Gates: Block Merges When Models Fail](https://www.openlayer.com/blog/cicd-eval-gates-block-merges-model-failure)
- [LangChain Docs — Cost tracking](https://docs.langchain.com/langsmith/cost-tracking)
- [Confident AI — What Is LLM Tracing? Traces, Spans, and Threads Explained](https://www.confident-ai.com/knowledge-base/guides/what-is-llm-tracing)
- [Langfuse — LLM-as-a-Judge](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)
- [Langfuse — Score Analytics](https://langfuse.com/docs/evaluation/evaluation-methods/score-analytics)
- [Langfuse — Evaluation of LLM Applications](https://langfuse.com/docs/evaluation/overview)
- [Arize Phoenix — GitHub](https://github.com/arize-ai/phoenix)
- [Arize — 14 best AI agent observability tools in 2026](https://arize.com/blog/best-ai-observability-tools-for-autonomous-agents-in-2026/)
- [W&B Weave Docs](https://docs.wandb.ai/weave)
- [W&B — LLM observability: Enhancing AI systems with W&B Weave](https://wandb.ai/onlineinference/genai-research/reports/LLM-observability-Enhancing-AI-systems-with-W-B-Weave--VmlldzoxMjY4MjMwNQ)
- [Laminar — GitHub](https://github.com/lmnr-ai/lmnr)
- [Laminar — Agent Observability: Tracing, Debugging, and Improving AI Agents](https://laminar.sh/article/agent-observability)
- [Laminar — Top 6 Agent Observability Platforms (2026)](https://laminar.sh/article/2026-04-23-top-6-agent-observability-platforms)
- [Datadog — Annotation Queues](https://docs.datadoghq.com/llm_observability/evaluations/annotation_queues/)
- [Agenta — Annotation Queues](https://agenta.ai/docs/changelog/annotation-queues)
- [Inngest — Durable Execution: The Key to Harnessing AI Agents in Production](https://www.inngest.com/blog/durable-execution-key-to-harnessing-ai-agents)
- [Vadim's blog — Durable Execution for LLM Agents: The Complete Guide](https://vadim.blog/durable-execution-llm-agents/)
- [Zylos Research — Durable Execution for AI Agent Runtimes: Checkpointing, Replay, and Recovery](https://zylos.ai/research/2026-04-24-durable-execution-agent-runtimes/)
- [Zylos Research — AI Agent Workflow Checkpointing and Resumability](https://zylos.ai/research/2026-03-04-ai-agent-workflow-checkpointing-resumability/)
- [Medium — LangSmith Tracing Deep Dive](https://medium.com/@aviadr1/langsmith-tracing-deep-dive-beyond-the-docs-75016c91f747)
- [LangChain — LangSmith vs Braintrust: Which AI Agent-Native Platform Fits Your Stack?](https://www.langchain.com/resources/langsmith-vs-braintrust)

**Sourcing caveat:** a large share of the comparison claims above (especially "X vs Y" framing) come from vendor-published articles, including several from Braintrust's own blog comparing itself to competitors. These are treated as MEDIUM confidence — directionally reliable for describing *what features exist*, less reliable for any claim about which vendor is "better." Where a claim appeared on only one source and could not be cross-checked against a second, it is called out inline (e.g., the LangSmith Engine "data flywheel" auto-clustering claim) rather than presented as settled fact.

---
*Feature research for: agent observability + evaluation platforms*
*Researched: 2026-09-08*
