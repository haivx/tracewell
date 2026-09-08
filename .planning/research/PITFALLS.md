# Pitfalls Research

**Domain:** Self-built agent tracing, eval harness, and checkpoint/recovery in TypeScript (Tracewell — minimum-Braintrust portfolio project)
**Researched:** 2026-09-08
**Confidence:** MEDIUM-HIGH (engineering pitfalls well-documented across current practitioner sources; eval-reliability and quality-gate numbers are grounded in cited 2025-2026 sources and a documented statistical formula, not just intuition; scoping pitfalls for this specific project are informed judgment applied to the constraints in PROJECT.md)

## Critical Pitfalls

### Pitfall 1: Trace context lost across async boundaries (parent-child nesting breaks silently)

**What goes wrong:**
Spans lose their parent, producing a trace that looks like a flat list of unrelated spans instead of a nested waterfall — the single most visually important part of the dashboard. This is the classic AsyncLocalStorage (ALS) failure: the "current span" context does not automatically follow execution across an `await` inside a detached callback, an event-emitter listener registered before the context was set, a `setTimeout`/`setImmediate` scheduled outside the ALS run, or (critically for a tool-calling agent) a `Promise.all` used for concurrent tool calls where each parallel branch needs its own child context under the same parent turn.

**Why it happens:**
Node's `AsyncLocalStorage.run()` only propagates to code that is causally *inside* the callback's continuation chain. Manually written tool-call loops (which this project deliberately uses instead of a framework) do not get this for free — every `await someTool()`, every `.then()`, and especially every `Promise.all([...toolCalls])` for parallel tool execution needs to either inherit the ALS store automatically (true for normal async/await chains in modern Node) or be given it explicitly (worker threads, detached timers, and pre-registered event listeners do not inherit it).

**How to avoid:**
- Wrap the entire agent turn (not each tool call) in one `AsyncLocalStorage.run()` so all descendants share the run's store, then create child spans by reading the current span from the store and setting it as parent explicitly — don't rely on ambient magic in a hand-rolled tracer.
- For parallel tool calls: capture the parent span **before** calling `Promise.all`, and pass it explicitly into each tool wrapper rather than expecting ALS to fork correctly — each of the N concurrent branches must open its own child span with the same explicit parent, not inherit a mutated "current span" that a sibling branch changed first (a shared mutable "current span" variable is a race condition under concurrency).
- Never use a plain module-level variable as "current span" — that is not concurrency-safe even single-threaded, because two overlapping async operations will stomp each other's value between microtask turns.
- Write one deliberate test early: kick off two tool calls in parallel from the same agent turn and assert both children point at the same parent span ID and neither leaks into the other's subtree.

**Warning signs:**
- Trace waterfall in the dashboard renders as siblings-at-root instead of nested, especially right after adding parallel tool calls.
- Two unrelated runs' spans appear interleaved under the same parent (context bled across runs, usually from a global/module-scope span reference reused between CLI invocations in the same process, e.g. in a test suite or a long-lived eval runner).
- Span count per run is right but tree depth is always 1.

**Phase to address:**
Phase 1 (Tracing) — this must be validated with an explicit parallel-tool-call test before the phase is called done, since the agent's tool loop (Phase 0) will already support multiple tool calls per turn once wired to modern tool-use APIs.

---

### Pitfall 2: Unbounded span attributes bloat the trace store and the dashboard

**What goes wrong:**
Spans that dump entire prompts, full file contents (from `read_file`), full lint/test output, or entire tool-call JSON blobs into span attributes. At small scale (10-20 eval cases, a handful of demo runs) this doesn't crash anything, but it makes traces slow to load in the dashboard, makes SQLite/Supabase rows enormous relative to their information value, and — more importantly for a portfolio project — makes the trace viewer look unpolished (a waterfall UI where every span's tooltip is a wall of text is a demo-killer, not a technical failure).

**Why it happens:**
The natural first instinct when instrumenting a span is "just log everything I have access to" — it's more info, seems safer for debugging. Nobody sets a byte budget until the payload is already visibly enormous.

**How to avoid:**
- Decide a per-attribute size cap up front (e.g. 2-4 KB) and truncate with a visible marker (`"...[truncated, 8214 bytes total]"`) rather than storing the whole thing.
- Store large blobs (full file contents, full model responses) at most once per span, referenced rather than duplicated — e.g. store the prompt once at the top-level agent-run span, and have child spans reference it by run ID instead of re-embedding it per tool call.
- Separate "structured, always-shown" attributes (tool name, duration, token counts, cost, status) from "large, on-demand" payloads (raw prompt, raw tool output) so the dashboard's default view stays cheap and a "show full payload" toggle stays optional.
- Treat this as a schema decision in Phase 1, not a later optimization — the trace schema is exactly the place to define "small structured fields" vs. "large optional blob fields," and OUT OF SCOPE already correctly says a high-performance trace store isn't the point — but that only holds if attribute size is bounded at write time.

**Warning signs:**
- A single trace's JSON export is measured in hundreds of KB for a run with fewer than ~10 tool calls.
- Dashboard trace list queries visibly slow down after only a few dozen runs.
- Copy-pasting a trace into a chat or doc for the write-up is unwieldy because it's dominated by repeated file contents.

**Phase to address:**
Phase 1 (Tracing) — bake truncation into the span-attribute helper from the first write, not retrofitted before the dashboard phase.

---

### Pitfall 3: Spans never closed on error paths (`try`/`catch` without `finally`)

**What goes wrong:**
A span is opened at the start of a tool call or agent step, and closed at the end of the "happy path" code — but if the tool throws, the code that would have closed and time-stamped the span is skipped. The resulting trace has a span with no end time (or, worse, a span that silently never gets flushed to the store), and the error itself may not be visibly attached to the span it happened in. This is especially damaging for the "bug appears, trace explains why" demo case (PROJECT.md's core value): the exact moment that most needs a clear trace — the failure — is the moment most likely to have a broken span.

**Why it happens:**
Instrumentation is usually added to the success path first ("wrap the call, log the result"), and error handling is bolted on afterward without revisiting span lifecycle. It's an easy category of bug to not notice, because everything looks fine until you specifically go looking at a trace for a failed run.

**How to avoid:**
- Standard pattern, non-negotiable for every span: open span → `try` the operation → on catch, record the exception on the span and set an error status → in `finally`, always call `span.end()` (or the local trace API's equivalent), regardless of success or failure.
- Write the span-wrapping helper once as a single utility (e.g. `withSpan(name, fn)`) that enforces this shape, and make every tool call and every agent step go through it — don't hand-write try/catch/span logic at each call site, or some call sites will inevitably skip the `finally`.
- Explicitly test this: force a tool to throw and assert (a) the span still appears in the trace with an end time and duration, (b) it's marked as errored, and (c) the parent span still closes and the trace as a whole is still valid/queryable (not left in a "run never finished" state).

**Warning signs:**
- Traces for failed runs are missing spans that succeeded runs have, or have spans with null/zero duration.
- The trace store has orphaned "open" spans that never got a matching end record.
- The demo case for "bug appears, trace explains why" only works when re-run enough times that the failure path is silently avoided.

**Phase to address:**
Phase 1 (Tracing) — this is the single highest-leverage test to write early, since the demo's entire narrative arc in Phase 4 depends on error-path traces being trustworthy.

---

### Pitfall 4: Cost calculated from a hardcoded or stale pricing table

**What goes wrong:**
Cost-per-span and cost-per-run numbers are computed from a pricing table baked into source code at whatever rates were current when that code was written. Anthropic pricing has changed across model generations, and even within a project lifetime of a few weeks a hardcoded table can silently drift from reality — the dashboard then shows confidently wrong dollar figures, which is worse than showing none, especially in an interview demo where a sharp interviewer might ask "is this number live?"

**Why it happens:**
Pricing tables feel like static reference data, so they get inlined as a constant object and forgotten. There's no natural trigger to notice it's gone stale, because the cost math still runs without error — it just produces plausible-looking wrong numbers.

**How to avoid:**
- Keep the pricing table in one versioned config file, separate from the tracing/cost logic, with a comment noting the date it was last checked against Anthropic's published pricing.
- Compute cost directly from the four token-usage counters the API actually returns per response (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) — never estimate tokens from string length, and never assume a flat rate that ignores the cache read/write multipliers (cache reads are priced far below regular input; cache writes are priced above it).
- Record the model ID actually used per span (not just assumed) — Claude's `response.usage` and the response's model field are the ground truth to key the pricing lookup off of, so a run against a different model than expected doesn't silently get priced at the wrong rate.
- For this project's scale, a single "confirm rates before the demo" checklist item is proportionate — no need to build a live-pricing-API integration, just don't hardcode-and-forget.

**Warning signs:**
- Cost figures in the dashboard don't move when you deliberately switch the agent to a cheaper/more expensive model.
- The pricing constant has no date/version comment and was written more than a few weeks before the demo.

**Phase to address:**
Phase 1 (Tracing), when cost fields are added to the trace schema — and re-verify once in Phase 4 right before recording/running the live demo.

---

### Pitfall 5: LLM-as-judge scores are unreliable in ways that specifically break a 10-20 case eval

**What goes wrong:**
LLM-as-judge scoring carries several well-documented, systematic biases, not just random noise: **position bias** (pairwise judges favor whichever response is shown first/second depending on the judge and task, strongly influenced by how close in quality the two options are — not just prompt length); **self-preference / family bias** (a judge favors outputs that resemble its own style, especially damaging if the judge and the agent-under-test are the same model family — a real risk here since both the code-review agent and its judge are likely to be Claude models); **verbosity bias** (judges reliably reward longer, more "authoritative-looking," well-formatted answers independent of correctness); and **non-determinism across runs** (the same case, judged twice, can get different scores even at low temperature, and judge-vs-judge or run-vs-run agreement is frequently far from perfect in current research). None of these show up as an obvious crash — they show up as a plausible-looking score that quietly measures the wrong thing.

**Why it happens:**
LLM-as-judge is attractive because it's cheap to set up and "sounds smart" compared to hard assertions, so it gets adopted for exactly the properties (code quality, review helpfulness) that are hardest to grade with a simple checker, which are also the properties most susceptible to these biases.

**How to avoid (current, cited mitigations):**
- For any pairwise/comparative judging, randomize which side is A/B per case, or score both orders and average — position bias is well-established across many judge models and worsens as quality gap narrows, so on close calls (which is exactly what a regression eval is trying to detect) it matters most.
- Explicitly instruct the judge not to reward length; keep the rubric about specific properties, not "which is better."
- Avoid using the exact model under test as its own judge; if budget allows, prefer a different model or a small jury for close calls — self-preference is documented even in 2025 EMNLP research as a real, measurable effect, distinct from genuine quality differences.
- Never tell the judge which output is the "reference" or "baseline" — label deference biases the score toward whichever side is marked authoritative.
- Use a concrete, checkable rubric with structured output (e.g. per-property JSON: `{correctness: bool, follows_style: bool, ...}`) rather than a single holistic 1-10 score — atomic checks are more reproducible and more diagnostic than one blended number.
- **Calibrate the judge against your own human labels before trusting it**: hand-score a handful of the 10-20 cases yourself, run the judge on the same cases, and check agreement. Well below roughly 90% agreement on clear-cut cases means the judge prompt needs iteration before its score should gate anything.
- Sanity-test the judge on known negatives: feed it an empty response, an "I don't know," and a confident wrong answer, and confirm it fails all three — this catches a judge that's too lenient before it ever touches the real eval set.
- Combine hard assertions (lint passed, correct bug line flagged, correct file identified) with the LLM judge for softer properties (review quality, explanation clarity) rather than leaning on the judge alone for pass/fail — this project's plan already does this, which is the right instinct; the risk is under-weighting the judge's contribution to noise when a case near the pass/fail boundary swings on judge variance alone.

**Warning signs:**
- Re-running the same eval case through the judge twice (same input, no code changes) produces different scores.
- Swapping which of two run outputs is "left" vs. "right" in a pairwise comparison flips the verdict.
- The judge consistently prefers the longer or more verbose of two otherwise-equivalent outputs.
- Judge score and hard-assertion pass/fail disagree on a large fraction of cases (a sign the judge is measuring something orthogonal to the ground truth).

**Phase to address:**
Phase 2 (Eval harness) — calibrate the judge against a hand-labeled subset before wiring it into the regression-comparison feature.

**Sources:** [Judging the Judges: Position Bias in LLM-as-a-Judge (ACL 2025)](https://aclanthology.org/2025.ijcnlp-long.18/); [Self-Preference Bias in LLM-as-a-Judge, EMNLP 2025](https://arxiv.org/pdf/2410.21819); [LLM-as-a-Judge: Why Frontier Models Fail 50%+ Bias Tests — Adaline](https://www.adaline.ai/blog/llm-as-a-judge-reliability-bias); Anthropic's own eval-audit checklist (bundled `claude-api` skill, `shared/evals/eval-audit.md`, §4 "When the grader is an LLM judge").

---

### Pitfall 6: Small eval dataset (10-20 cases) treated as statistically meaningful when it isn't — and quality gates built on top of that illusion

**What goes wrong:**
A 10-20 case eval set combined with a non-deterministic judge produces a pass-rate whose natural sampling noise is *larger* than the score differences a quality gate is meant to detect. A pass-rate confidence interval half-width is roughly `1/sqrt(n × R)` where `n` is case count and `R` is reps per case — at 15 cases with 1 rep, that's roughly ±26 percentage points; even at 20 cases with 2 reps, it's still roughly ±16 points. A gate set to "warn if score drops more than 5-10%" against that noise floor will fire on random judge variance as often as on a real regression — the exact "threshold gates that are noisy" trap named in this research brief.

**Why it happens:**
It's intuitive to treat "18/20 passed" as a precise measurement rather than a point estimate with a wide error bar, especially because a percentage looks precise. The temptation to overfit is also real: with so few cases, it's easy to unconsciously tune the agent/prompt until it passes exactly those 10-20 cases, producing a number that looks great but doesn't generalize — classic small-n overfitting.

**How to avoid:**
- Explicitly compute and report the noise floor (the CI half-width above) next to any pass-rate, both in the eval harness output and — importantly for the interview teardown — as a documented, named limitation rather than something quietly glossed over. Being explicit about *why* the threshold is set where it is, and what its false-positive/false-negative rate looks like given the noise floor, is itself a strong technical-depth signal for a portfolio piece.
- Run multiple reps per case (2-3, budget permitting) rather than one — reps shrink the noise floor faster and cheaper than adding more cases for a small hand-built fixture set, and a paired-difference design (same cases, same judge, comparing run A vs run B on identical inputs) is more sensitive than comparing two independent aggregate scores.
- Prefer hard assertions wherever the ground truth allows it (lint/test pass-fail is already the plan) — deterministic checks have zero judge-variance noise, so lean on them for anything with an objective answer and reserve the judge for genuinely subjective properties, minimizing how much of the final score rides on the noisy component.
- See Pitfall 7 for how to translate this into an actual gating strategy rather than a single brittle threshold.

**Warning signs:**
- Re-running the full eval twice against the identical code produces a different score.
- A deliberately-introduced "obviously worse" agent (e.g. a version that skips a tool) scores within noise of the "good" version.
- The eval score improved suspiciously quickly early on and then plateaued exactly at 100% on all 10-20 cases — a signature of overfitting to the fixture set rather than genuine improvement.

**Phase to address:**
Phase 2 (Eval harness) for computing and surfacing the noise floor; Phase 3 (quality gate) for turning it into a gating decision (see Pitfall 7).

**Sources:** [Measuring all the noises of LLM Evals (2025)](https://arxiv.org/pdf/2512.21326); [Stochasticity in Agentic Evaluations — Intraclass Correlation](https://arxiv.org/pdf/2512.06710); Anthropic's eval-audit checklist §5 "Can it detect the change you're after?" (bundled `claude-api` skill), which gives the `1/sqrt(n·R)` noise-floor formula directly.

---

### Pitfall 7: A single hard threshold gate is the wrong gating mechanism for this dataset size

**What goes wrong:**
A gate defined as "fail if score < 80%" (or "drops more than X points") is a coin flip at this dataset size — see Pitfall 6. Teams that hit this build a gate, watch it cry wolf a few times, and either disable it (losing the safety net entirely) or keep tightening the threshold until it never fires (also losing the safety net, just more quietly).

**Why it happens:**
A single scalar threshold is the simplest thing to implement and matches the mental model of a CI test ("pass" or "fail"), but CI-test intuition assumes near-zero measurement noise, which does not hold for a judge-scored eval over a handful of cases.

**How to avoid (the right way to gate under these conditions):**
- **Gate on hard-assertion regressions primarily, and treat the LLM-judge component as informational/directional rather than a hard pass/fail line** — e.g. "any previously-passing seeded-bug case that now fails its hard assertion is a hard fail; a judge-score drop is a warning to look at, not an auto-fail," matching this project's own instinct to combine hard assertions with a judge, taken to its logical conclusion for gating specifically.
- **Report the comparison as a paired diff with the noise floor attached**, not a bare pass/fail: "score changed from 17/20 to 15/20; noise floor is ±3 cases at this rep count, so this is within/outside expected variance." This is more honest and, again, is a better artifact for the teardown than a green/red checkmark.
- **Widen the gate's threshold to sit outside the computed noise floor**, not at an arbitrary round number — if the noise floor is ±15 points, a gate at "-5 points" is meaningless noise-chasing; a gate at "-20 points, or any newly-failing hard assertion" is actually discriminating signal from noise.
- **Increase reps for judged cases near the decision boundary** rather than trusting a single judge call — a cheap "if hard-assertions are unchanged and judge score moved but not by more than the noise floor, re-run the judge 2-3 more times on just the affected cases before deciding" step directly addresses non-determinism without paying for extra reps on every case, every time.
- Explicitly do **not** build a fully general statistical-significance testing framework for this — that's disproportionate for a portfolio project. A documented noise-floor-aware threshold plus "hard assertions gate, judge score informs" is the right-sized solution; state the simplification in the teardown rather than hiding it.

**Warning signs:**
- The gate fires on every run, including "no code changes" no-op re-runs.
- The team has already manually overridden/ignored the gate more than once.
- Nobody can explain what score-delta the gate is actually built to catch, in plain language.

**Phase to address:**
Phase 3 (checkpoint/recovery + quality gate) — this phase's brief already frames the work as "determinism at the edge of a non-deterministic system," which is exactly the right frame; the gate design should be treated as a first-class design decision worth its own short section in the teardown, since it's a genuinely interesting engineering tradeoff to discuss in an interview.

---

### Pitfall 8: Non-idempotent tool replay on checkpoint resume

**What goes wrong:**
On resume, a naive checkpoint/recovery implementation re-executes the step that was in-flight when the run was interrupted — which, if that step was a side-effecting tool call (e.g. `run_linter`, `run_tests`, or any tool that writes/modifies files), runs it a second time. For a read-only code-review agent this is lower-stakes than for a general agent, but it's still a real bug class worth designing around deliberately, and it's exactly the kind of subtlety that makes for a strong teardown section (Anthropic's own LangGraph-adjacent research explicitly flags "resume re-runs the node; it does not continue from the next line" — idempotency is non-negotiable for any side-effecting step, not an edge case).

**Why it happens:**
The simplest checkpoint design saves "we were about to run step N" and, on resume, just runs step N again — which is correct for pure/read-only steps but wrong for anything with a side effect, and the two categories are easy to conflate when the checkpoint format doesn't distinguish them.

**How to avoid:**
- Classify every tool in the agent's tool surface as idempotent (safe to re-run: `read_file`, `search_codebase`) vs. non-idempotent (has a side effect or non-trivial cost: `run_tests`, `run_linter` if it has side effects like auto-fix flags). Only re-run idempotent tools blindly on resume.
- For non-idempotent tools, checkpoint the **result**, not just "we were about to call it" — if a checkpoint is taken *after* a tool call completes and *before* the next model call, resume never needs to replay the tool at all, only re-send its already-known result to the model. This is the simplest fix and fits this project's scale well: checkpoint at tool-call boundaries, not mid-call.
- If a checkpoint must be taken mid-tool-call (rare, and avoidable at this project's scale by just checkpointing at boundaries), record enough to detect "this already ran" on resume (e.g. a run ID/hash) so a retry can short-circuit rather than blindly re-executing.
- Given the code-review agent's tools are mostly read-only or naturally idempotent (`read_file`, `search_codebase`, `run_linter`, `run_tests` all produce the same result if run twice on unchanged code), this pitfall is lower-severity for this specific project than for a general agent — but it's worth stating that reasoning explicitly in the checkpoint design rather than leaving it implicit, since "why doesn't this need distributed-lock-level idempotency guards" is a legitimate interview question with a good, specific answer here.

**Warning signs:**
- Resume logic re-invokes a tool without checking whether its result is already recorded from before the interruption.
- Test/lint output counts differ between a clean run and a run that was interrupted-and-resumed partway through, on identical code.

**Phase to address:**
Phase 3 (checkpoint/recovery) — decide the checkpoint boundary (after tool result, before next model call) as the first design decision of this phase, before writing persistence code.

**Sources:** [Why Checkpoints Aren't Durable Execution — Diagrid](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows); [The Hidden Replay Risk in LangGraph — Medium](https://medium.com/@mehul_parmar/the-hidden-replay-risk-in-langgraph-how-durable-execution-can-burn-you-1d966141e71a).

---

### Pitfall 9: Checkpoint state that cannot be rehydrated, or drifts from the trace

**What goes wrong:**
Two related failures: (1) the checkpoint serializes something that can't be faithfully reconstructed later — e.g. a live handle, a class instance with methods, a closure, or a reference to something only valid within the original process's memory (an open file descriptor, a connection) — so deserializing it on resume either throws or silently produces a broken stand-in object; (2) the checkpoint and the trace store diverge — the checkpoint says the agent was "about to call tool X" but the trace shows tool X already completed (or vice versa), because the two are written by different code paths that aren't kept in lockstep, so which one is "true" becomes ambiguous exactly when it matters most (after a crash).

**Why it happens:**
Checkpointing and tracing are usually built as two separate concerns (they're even two separate phases in this roadmap), so it's natural to write each with its own notion of "the current step" rather than deriving both from one shared source of truth.

**How to avoid:**
- Checkpoint only plain, serializable data: message history as plain objects/JSON, tool call inputs/outputs as already-serialized JSON (which they must be to send over the API anyway), and simple state flags — never a live object, class instance, or anything requiring custom deserialization logic beyond `JSON.parse`.
- Derive the checkpoint's "current step" pointer from the same event stream that produces trace spans, rather than maintaining two independent step counters — e.g. the checkpoint records the ID of the last trace span that was fully closed, so resume logic can always ask "what's the last known-good span?" instead of trusting a separately-maintained state machine that could have drifted.
- On resume, validate the checkpoint against the trace before trusting it: if the checkpoint claims step N is done but the trace's last closed span is for step N-1, trust the trace (or at minimum, surface the mismatch loudly rather than silently picking one) — this single validation check is cheap to write and catches the entire divergence class.
- Version the checkpoint schema explicitly (even a bare `schemaVersion: 1` field) so a checkpoint written by an earlier version of the agent code doesn't get blindly deserialized against a changed schema later in the project — a real risk over a multi-week part-time project where the agent's message/state shape is likely to evolve between sessions.

**Warning signs:**
- Deserializing an old checkpoint throws or produces `undefined` fields that used to be populated.
- Resume "succeeds" but the model's next response references information it shouldn't have (or is missing information it should have) relative to what the trace shows actually happened.

**Phase to address:**
Phase 3 (checkpoint/recovery) — design the checkpoint format explicitly against "does this reconstruct with `JSON.parse` alone" and cross-check it against the trace schema from Phase 1 before writing persistence logic.

---

### Pitfall 10: Resuming into a context the model can no longer make sense of

**What goes wrong:**
Even when the checkpoint deserializes cleanly and every tool result is faithfully restored, the model itself can be confused on resume for reasons that have nothing to do with serialization: a trailing tool call with no matching result (from a crash mid-tool-call) left dangling in the message history and sent back to the model, which is exactly the malformed shape most tool-use APIs reject or mishandle; a resumed conversation whose last message is a tool result but which never actually reached a natural "assistant turn" boundary, confusing the model about whose turn it is; or simply a long-enough gap between interruption and resume that "resuming" produces a response that no longer matches what the user (or eval case) actually needs, because the state the model reasons over is stale relative to reality (e.g. the fixture repo's files changed between the crash and the resume, but the trace/checkpoint has the old file contents cached in earlier tool results).

**Why it happens:**
It's easy to think of "resume" as "replay the message array and continue," but the message array is only valid to replay if it's in a state the API can accept and the model can reason from coherently — a mid-tool-call crash produces neither by default.

**How to avoid:**
- On crash, before persisting the final checkpoint, prune any trailing incomplete tool call that has no matching tool result — either discard the dangling tool_use block entirely (safe, since it never got a result to reason from anyway) or replay just that one tool call fresh and attach its result before resuming the rest of the conversation.
- Only checkpoint at clean turn boundaries where possible (see Pitfall 8's "checkpoint after tool result, before next model call") — this sidesteps most of the dangling-call problem structurally rather than needing cleanup logic.
- For the demo specifically: pick a seeded-bug case where the interruption point is deterministic and well-understood (e.g. manually kill the process right after a specific tool call completes) rather than relying on a naturally-occurring, unpredictable failure — a controlled, reproducible resume scenario is both easier to build reliably and a better demo than trying to catch a real random failure live.
- If the fixture repo's state could plausibly change between interruption and resume in this project's context (unlikely, since it's a static seeded-bug fixture, but worth stating explicitly), don't cache full file contents from `read_file` across a resume boundary without at least a staleness check — re-reading is cheap and safe here specifically because the tools are idempotent and read-only.

**Warning signs:**
- Resume occasionally produces an API error about malformed message structure (unmatched tool_use/tool_result pairs).
- The model's first response after resume references something inconsistent with what actually happened before the crash.

**Phase to address:**
Phase 3 (checkpoint/recovery) — specifically worth a deliberate test: force a crash mid-tool-call (not just between turns) and verify resume produces a valid, sensible continuation, not just "doesn't throw."

**Sources:** practitioner analysis of long-running agent resume ("trailing incomplete calls must be pruned to avoid confusing the model with dangling references") and context-drift-on-resume failure patterns; [Why AI Agents Break: A Field Analysis of Production Failures — Arize](https://arize.com/blog/common-ai-agent-failures/).

---

### Pitfall 11: Gold-plating the dashboard instead of shipping the demo narrative

**What goes wrong:**
The dashboard phase (Phase 4) is the most visually seductive part of the project — it's the part an interviewer actually *looks at* — which makes it the easiest place to over-invest: polishing chart animations, building a generic "explore any trace" UI, adding filters and views nobody will use in the one prepared demo case, or reaching for design/component-library polish that doesn't serve the specific "bug → trace → eval catches regression" narrative this project is built around. Every hour spent here is an hour not spent on the write-up, which this project's own constraints flag as the single biggest risk (stalling before the architecture teardown is written).

**Why it happens:**
UI work has fast, satisfying visual feedback loops (every change is immediately visible), which makes it easy to keep going "just a little more" in a way that backend/infra work rarely tempts you into — especially for someone whose stated goal is repositioning into AI engineering rather than frontend, where the dashboard is explicitly a supporting artifact, not the point (PROJECT.md: "spend learning budget on the infra layer, not the UI layer").

**How to avoid:**
- Write down, before starting Phase 4, the exact three or four dashboard views the demo narrative requires (trace waterfall, run comparison diff, cost/token breakdown — already scoped in PROJECT.md) and treat anything beyond that list as explicitly out of scope for this milestone, not "nice to have if there's time."
- Build the dashboard against the one prepared demo case first, not a generic "works for any trace" abstraction — generalize only if doing so is *free*, never as a goal in itself.
- Time-box the dashboard phase explicitly (e.g. "two sittings, then whatever's built is what ships") and schedule the teardown write-up to start before the dashboard is "finished," so the write-up isn't perpetually waiting on a dashboard that keeps growing scope.
- Remember PROJECT.md already explicitly deferred the hardest temptation (eval-scores-over-time trend view) — hold that line; a trend chart is exactly the kind of visually appealing feature that's easy to justify "just for polish" but adds no signal from only a handful of runs.

**Warning signs:**
- Time spent on the dashboard visibly exceeds time spent on tracing + eval + checkpoint combined.
- The dashboard supports views/filters that don't appear anywhere in the prepared demo script.
- The teardown document has not been started by the time the dashboard is "80% done."

**Phase to address:**
Phase 4 (Dashboard + demo + teardown) — the fix here is almost entirely a scoping/discipline issue, not a technical one; the "Out of Scope" section of PROJECT.md already provides the right guardrails, the risk is drifting past them mid-phase.

---

### Pitfall 12: Building storage sophistication nobody needs

**What goes wrong:**
Over-engineering the persistence layer — e.g. building an abstraction layer to support swapping SQLite/Supabase/a hypothetical future store, adding indexes and query optimizations for a trace store that will hold, at most, a few dozen runs from a single-user demo, or implementing anything resembling Brainstore-class trace-store engineering (explicitly called out as out of scope in PROJECT.md). This is the classic "build for the scale you wish you had" trap, and it's a particularly easy one to fall into on an infra-focused portfolio project, because storage-layer sophistication *feels* like the kind of depth an interviewer would respect — but depth that doesn't serve this project's actual scale reads as wasted effort once questioned, whereas a simple, correctly-reasoned-about SQLite schema with a clearly stated migration plan reads as good judgment.

**Why it happens:**
"What if this needs to scale later" is a reasonable instinct in production engineering, but it's miscalibrated for a project whose entire point (per PROJECT.md) is "the learning is in the schema, not the storage engine," and whose known migration point (SQLite → Supabase at the dashboard phase, already an approved Key Decision) is a one-time, planned event, not a scaling problem to solve preemptively.

**How to avoid:**
- Design the trace/eval/checkpoint schemas to be storage-engine-agnostic at the level of "these are the tables/columns and their types," not at the level of building a repository-pattern abstraction layer with pluggable backends — the planned single migration (SQLite → Supabase) is a one-time port, not a reason to build a permanent abstraction.
- Resist adding indexes, caching layers, or query optimizations until an actual, observed slowness shows up in the demo-scale dataset (which, per PROJECT.md's own numbers — 10-20 eval cases, a handful of comparison runs — is very unlikely to ever materialize).
- When the "what if this needs to scale" thought appears, write it down as a one-line note in the teardown ("at production scale you'd want X, e.g. a dedicated trace store like Brainstore, sharded by run ID") rather than building it — demonstrating awareness of the scaling path is exactly as valuable in an interview as building it, and far cheaper.

**Warning signs:**
- Time spent on database abstraction/repository patterns exceeds time spent on the actual trace schema design.
- Code exists to support a storage backend that is never actually used (e.g. a config flag for a store that was never built).

**Phase to address:**
Phase 1 (Tracing, where the store is first built) and Phase 3 (checkpoint persistence) — both are natural entry points for this trap; the guardrail is re-reading PROJECT.md's Out of Scope section before starting either.

---

### Pitfall 13: The project stalls before the architecture teardown is written

**What goes wrong:**
This is explicitly named in the project's own constraints as the single biggest risk, and it's worth treating with the same rigor as a technical pitfall rather than a vague "stay motivated" note. The concrete failure mode: all four engineering phases get built (agent, tracing, eval, checkpoint/dashboard), the live demo works, and then the project quietly stops before the write-up — the actual interview artifact — gets produced, because writing is a different kind of work than building, it has no "does it run" feedback loop, and it's easy to keep finding "one more thing" to build/polish instead of switching modes to writing.

**Why it happens:**
Building has continuous positive feedback (tests pass, the demo runs) that writing lacks; and because the codebase is right there and familiar, "improve the code a bit more" is always the path of least resistance compared to the colder-start activity of writing prose about it. For a part-time, interruptible, multi-week project, this risk compounds every session it's deferred, because the mental context needed to write a good teardown ("why did I make this tradeoff") decays between sessions faster than code does.

**How to avoid:**
- Start the teardown document in Phase 0 or Phase 1, as a running log of decisions and their rationale (this project's own Key Decisions table in PROJECT.md is already a good seed for this) — not as a single write-up attempted cold at the very end after everything else is "done."
- Treat "one paragraph in the teardown about what this phase taught" as part of the definition of done for every phase, not an activity reserved for Phase 4 — this converts the biggest risk into a habit distributed across the whole project rather than one large, avoidable task at the end.
- Explicitly timebox Phase 4 to include teardown-writing time from the start (not "build dashboard, then somehow also find time to write"), and consider writing the teardown's outline/skeleton before the dashboard is built, so dashboard work is visibly in service of specific sections that already exist.
- If momentum stalls mid-project, the fallback is to write the teardown for whatever phases *are* done rather than waiting for full completion — a well-written teardown covering three finished phases is a far better outcome than an unfinished fourth phase and no writing at all.

**Warning signs:**
- Multiple work sessions pass with commits only to phase code, none to the teardown doc.
- The teardown doc doesn't exist yet by the time Phase 2 or 3 is complete.
- The plan for "when to write the teardown" is "after everything else is done" rather than something with its own scheduled sessions.

**Phase to address:**
Every phase — this is the one pitfall that isn't fixed by a single phase's design decision, but by a standing process rule applied from Phase 0 onward (start the teardown early, update it every phase).

---

### Pitfall 14: Demo fragility from a live, non-deterministic model call

**What goes wrong:**
The prepared demo case ("bug appears, trace explains why, eval catches the regression") depends on a live Claude API call producing a specific, expected output during a screen-share — but LLM outputs are not perfectly deterministic even at low temperature, and a live call also introduces real failure surfaces that have nothing to do with the actual work being demonstrated: API latency spikes, a rate limit hit at the worst moment, a transient 5xx/529, or the model taking a slightly different tool-call path than it did in rehearsal and not tripping the seeded bug the same way. Any of these turns a technical demo into a live debugging session in front of an audience — the single worst outcome for an interview artifact.

**Why it happens:**
A live demo *feels* more impressive and honest than a recorded one ("look, it's really happening right now"), so there's a natural pull toward doing it live even though the actual content being demonstrated (trace structure, eval scoring, checkpoint resume) doesn't require live inference to be convincing.

**How to avoid:**
- Record a video walkthrough of the full "bug → trace → eval regression" narrative as the primary artifact, and reserve a live component (if any) for something that doesn't depend on a fresh model call succeeding a specific way — e.g. live-navigating a dashboard built from a pre-recorded trace, rather than live-triggering the agent.
- If a live element is wanted for interactivity, pre-run and cache the exact demo scenario's trace/eval output ahead of time as a fallback, and script the demo to fall back to "let me show you the pre-recorded run of exactly this" if the live call misbehaves — never let the demo's success depend entirely on one specific live API response landing exactly as rehearsed.
- Pick a seeded bug for the demo that the agent finds *reliably* across multiple manual dry runs (not one that worked once) — run the demo scenario several times beforehand and only commit to it as "the" demo case once it has shown consistent behavior across repeats.
- Retry policy for the demo specifically: wrap the live portion (if kept) with the SDK's automatic retry behavior for 429/5xx/network errors, and set a generous `max_tokens` so a mid-response truncation doesn't derail the narrative (see Pitfall 15) — small robustness investments here pay for the whole audience-facing moment.

**Warning signs:**
- The demo has never been run start-to-finish more than once or twice before the real presentation.
- No fallback plan exists for "what if the live call doesn't reproduce the bug this time."
- The demo's most impressive moment depends on the model doing something creative/non-deterministic rather than something scripted to be reliable.

**Phase to address:**
Phase 4 (Dashboard + demo + teardown) — decide "recorded vs. live" deliberately as a demo-design decision, not a default; rehearse the chosen demo case at least 3-5 times before it's considered final.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|-----------------|------------------|
| Storing raw prompts/file contents unbounded in span attributes | Faster to write, "just log everything" | Bloated store, slow dashboard, unreadable trace exports | Never — cap at write time, even in early prototyping |
| Single scalar quality-gate threshold | Trivial to implement | Gate is noise-dominated at this dataset size, either cries wolf or is silently disabled | Only as a documented placeholder before the noise-floor-aware gate (Pitfall 7) is built |
| Checkpointing mid-tool-call instead of at tool-result boundaries | Slightly finer-grained recovery | Requires idempotency/dedup machinery to avoid double side effects | Only if a specific tool call is extremely long-running and the boundary-only approach loses too much progress on interruption — unlikely at this project's scale |
| Skipping judge calibration against hand-labels | Saves an hour of manual labeling | Judge scores may not track real quality, undermining every downstream comparison and the quality gate | Never for the final eval — acceptable to skip only during very early prototyping of the judge prompt itself |
| Hardcoded pricing constants | Quick to ship | Silently wrong cost figures after any pricing change | Acceptable short-term with a dated comment; never acceptable un-reviewed before the demo |

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|-----------------|-------------------|
| Claude API tool use | Splitting parallel tool results across multiple user messages | Return **all** `tool_result` blocks for one assistant turn's `tool_use` blocks in a **single** user message — splitting them silently trains the model to stop making parallel calls |
| Claude API tool use | Assuming a `tool_use` block is always complete | Check `stop_reason === "max_tokens"` before trusting a `tool_use` block — a truncated response can end mid-tool-call with an unusable, incomplete input; retry with higher `max_tokens` rather than attempting to execute a partial tool call |
| Claude API tool use | Forcing exactly one tool call via `tool_choice: {type: "tool", name: ...}` and hardcoding that assumption elsewhere in the loop | Default to `{type: "auto"}` and design the loop to naturally handle zero, one, or many tool_use blocks per turn — do not special-case "exactly one" |
| Claude API errors | Catching one broad exception class for all API errors | Chain typed exceptions most-specific-first (e.g. `RateLimitError` before `APIStatusError` before `APIConnectionError` in TypeScript) so retryable (429/5xx/network) and non-retryable (4xx) failures are handled differently |
| Claude API rate limits | No backoff, or a zero-delay retry loop when running the eval repeatedly | Rely on the SDK's built-in exponential backoff for 429/5xx (default `max_retries: 2`), and when running the full 10-20 case eval multiple times (once per code change), be mindful that each pass is real spend — batch runs where possible |
| Claude API cost control for repeated evals | Running every eval pass at full interactive pricing | Use the **Message Batches API** (50% off every token, including cache reads/writes) for eval runs, since eval passes are not latency-sensitive — results arrive within 24 hours (usually much faster) and this is the single largest lever specifically suited to "run the same 10-20 cases repeatedly" |
| Claude API prompt caching | Not caching the system prompt / tool schema prefix across the 10-20 eval cases, which share nearly all of it | Add an explicit `cache_control` breakpoint on the static system prompt + tool definitions prefix — since every eval case reuses the same prefix, this alone can substantially cut eval-run cost, and it's a "free win" (no quality tradeoff) |
| SQLite → Supabase migration (Key Decision already made) | Designing SQLite-specific schema quirks (e.g. relying on SQLite's flexible typing) that don't port cleanly | Write schema and queries against the common subset both support from day one, so the planned migration at the dashboard phase is mechanical rather than a mini-rewrite |

## Performance Traps

Patterns that work at small scale but fail as usage grows. (Included for completeness; most have generous scale thresholds given this project's explicit small-scale scope.)

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|-----------------|
| Unbounded span attribute size | Dashboard trace view feels sluggish, exports are huge | Truncate at write time (Pitfall 2) | Noticeable well before 100 runs given file-content-sized attributes |
| No indexes on trace store queries (run ID, span parent ID) | Trace waterfall queries slow as run count grows | Add basic indexes once query patterns are known — cheap, low-risk, unlike broader storage sophistication (Pitfall 12) | Unlikely to matter below a few hundred runs at this project's scale, but trivial to add early since it's not "sophistication," just correctness |
| Full eval re-run (all cases, all reps) on every small code tweak | Burns API budget quickly during iterative development | Support running a subset (e.g. just the affected/changed cases) during iteration, full set only before a checkpointed comparison | Matters immediately given a fixed personal API budget, even though the dataset itself is small |

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Agent's file-reading tool (`read_file`) allows path traversal (`../../etc/passwd`, symlinks, absolute paths outside the fixture repo) | Model-directed reads escape the intended project root | Resolve the model-supplied path to its canonical form and verify it stays within the fixture repo root before reading; reject `..`, symlinks that escape, and absolute paths outside the root |
| Agent's `run_linter`/`run_tests` tools execute shell commands built from model-influenced input | Command injection if any part of the command string is model-controlled without validation | Treat tool-provided arguments as untrusted; use an allowlist of exact commands/scripts to run rather than string-interpolating model output into a shell command |
| API key handling in a portfolio repo that may be shared/screen-shared | Leaked key if committed or visible during a live demo | Load from environment variable only, never hardcode; double-check terminal/env output isn't visible during the screen-shared demo |
| Trace/eval data containing fixture-repo file contents committed to a public portfolio repo | Low risk here since fixtures are synthetic seeded bugs, but worth a deliberate check | Confirm the fixture repo contains no real/sensitive code before making the portfolio repo public |

## UX Pitfalls

Common user experience mistakes in this domain (the "user" here is largely the interviewer/viewer of the demo and dashboard).

| Pitfall | User Impact | Better Approach |
|---------|-------------|-------------------|
| Trace waterfall shows raw, untruncated JSON blobs in the default view | Viewer's eyes glaze over; the interesting structure (nesting, timing, cost) is buried | Show structured summary fields by default (name, duration, cost, status), full payload behind an explicit expand/toggle |
| Run comparison diff shows a single aggregate score with no context | Viewer can't tell if a score change is a real regression or noise | Show the paired per-case diff and the noise floor alongside the headline number (ties directly to Pitfall 6/7) |
| Dashboard has many views/filters but the demo only uses three | Viewer wonders what the rest is for, or the presenter fumbles navigating unused UI live | Keep the dashboard's visible surface area matched to what the demo script actually uses (Pitfall 11) |

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Tracing:** Often missing error-path span closure — verify by forcing a tool to throw and checking the resulting trace has a properly closed, error-flagged span (Pitfall 3).
- [ ] **Tracing:** Often missing parent-child correctness under concurrency — verify by running two tool calls in parallel from one turn and checking both share the correct parent (Pitfall 1).
- [ ] **Eval harness:** Often missing judge calibration — verify by hand-scoring a subset of cases and checking judge agreement before trusting the judge's score anywhere else (Pitfall 5).
- [ ] **Eval harness:** Often missing noise-floor reporting — verify the harness prints/logs a confidence interval or noise estimate alongside any pass-rate, not just the raw number (Pitfall 6).
- [ ] **Checkpoint/recovery:** Often missing a mid-tool-call crash test — verify by forcing a crash *during* a tool call (not just between turns) and confirming resume produces a valid, coherent continuation (Pitfall 10).
- [ ] **Quality gate:** Often missing an explanation of *why* the threshold is set where it is — verify the gate's threshold is derived from (or at least checked against) the computed noise floor, not picked arbitrarily (Pitfall 7).
- [ ] **Cost tracking:** Often missing verification against current rates — verify the pricing table has a recent "checked against docs on [date]" note before the demo (Pitfall 4).
- [ ] **Demo:** Often missing a rehearsed, repeatable run — verify the exact demo scenario has been run successfully at least 3-5 times before it's the plan (Pitfall 14).
- [ ] **Teardown:** Often missing until the very end — verify the teardown document exists and has content by the end of Phase 1, not just Phase 4 (Pitfall 13).

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|----------------|------------------|
| Lost parent-child trace context (Pitfall 1) | LOW | Refactor the span-creation helper to take an explicit parent argument everywhere, removing reliance on ambient "current span" state; add the parallel-tool-call test retroactively |
| Unbounded attributes already in the store (Pitfall 2) | LOW | Add truncation to the write path going forward; optionally run a one-time cleanup pass over existing rows, but not required given the small data volume at this scale |
| Judge found to be poorly calibrated after cases are already scored (Pitfall 5) | MEDIUM | Re-score the affected cases with an improved judge prompt; since the dataset is only 10-20 cases, a full re-run is cheap — this is one of the advantages of staying small |
| Quality gate found to be noise-dominated after being relied on for a while (Pitfall 6/7) | LOW | Recompute the noise floor, widen the threshold or switch to the hard-assertions-gate/judge-informs split (Pitfall 7); no data loss, just a config/logic change |
| Checkpoint format found to be unrehydratable after a schema change (Pitfall 9) | MEDIUM | Add a `schemaVersion` field going forward if missing; for already-broken old checkpoints, they're disposable at this project's scale — just discard and re-run rather than writing migration logic for throwaway demo data |
| Demo found fragile close to presentation time (Pitfall 14) | LOW | Fall back to a recorded walkthrough of a previously-successful run; this is why rehearsing early (not the night before) matters — it leaves time to make this call calmly |
| Project stalled before teardown (Pitfall 13) | HIGH (time cost) but fully recoverable | Write the teardown now, for whatever is actually built and working — a smaller, honest, finished teardown beats a larger, unfinished project every time in an interview context |

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|-------------------|----------------|
| 1. Lost parent-child trace context | Phase 1 (Tracing) | Parallel-tool-call test asserts shared correct parent |
| 2. Unbounded span attributes | Phase 1 (Tracing) | Inspect a real trace export's byte size; confirm truncation markers appear on large payloads |
| 3. Spans not closed on error | Phase 1 (Tracing) | Forced-throw test confirms span closes, is marked errored, duration is set |
| 4. Stale/hardcoded pricing | Phase 1 (Tracing), re-verify Phase 4 | Pricing config has a dated "last checked" note; cost changes when model is swapped |
| 5. LLM-judge unreliability | Phase 2 (Eval harness) | Judge-vs-hand-label agreement measured on a subset before trusting judge scores elsewhere |
| 6. Small-n statistical illusion | Phase 2 (Eval harness) | Harness reports a noise-floor/CI figure alongside every pass-rate |
| 7. Noisy threshold gate | Phase 3 (Quality gate) | Gate logic documented against the measured noise floor; hard-assertion regressions gate separately from judge-score drift |
| 8. Non-idempotent tool replay | Phase 3 (Checkpoint/recovery) | Interrupt-and-resume test confirms a side-effecting tool is not re-invoked with duplicated effect |
| 9. Unrehydratable/divergent checkpoint state | Phase 3 (Checkpoint/recovery) | Checkpoint round-trips through `JSON.parse`/`JSON.stringify` cleanly; cross-checked against trace's last closed span on resume |
| 10. Resuming into an incoherent context | Phase 3 (Checkpoint/recovery) | Mid-tool-call crash test confirms resume produces a valid, sensible continuation (no dangling tool_use blocks) |
| 11. Dashboard gold-plating | Phase 4 (Dashboard/demo/teardown) | Dashboard's built views map 1:1 to the prepared demo script; nothing extra |
| 12. Storage over-engineering | Phases 1 & 3 | No abstraction layer beyond what the one planned SQLite→Supabase migration needs |
| 13. Stalling before the teardown | All phases | Teardown doc exists and has content by end of Phase 1; updated every phase thereafter |
| 14. Demo fragility from live non-determinism | Phase 4 (Dashboard/demo/teardown) | Demo scenario rehearsed successfully 3-5+ times; fallback recorded version exists |

## Sources

- [Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge (ACL 2025)](https://aclanthology.org/2025.ijcnlp-long.18/)
- [Self-Preference Bias in LLM-as-a-Judge (EMNLP 2025)](https://arxiv.org/pdf/2410.21819)
- [LLM-as-a-Judge: Why Frontier Models Fail 50%+ Bias Tests — Adaline](https://www.adaline.ai/blog/llm-as-a-judge-reliability-bias)
- [Measuring all the noises of LLM Evals (2025)](https://arxiv.org/pdf/2512.21326)
- [Stochasticity in Agentic Evaluations: Quantifying Inconsistency with Intraclass Correlation](https://arxiv.org/pdf/2512.06710)
- [evalstats — statistical analysis for LLM evals at small sample sizes](https://github.com/ianarawjo/evalstats)
- [How to Propagate Trace Context Across Async Boundaries — OneUptime](https://oneuptime.com/blog/post/2026-02-06-propagate-trace-context-async-boundaries/view)
- [OpenTelemetry Context docs — async boundary propagation](https://opentelemetry.io/docs/languages/js/context/)
- [Why Checkpoints Aren't Durable Execution — Diagrid](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)
- [The Hidden Replay Risk in LangGraph — Medium](https://medium.com/@mehul_parmar/the-hidden-replay-risk-in-langgraph-how-durable-execution-can-burn-you-1d966141e71a)
- [Why AI Agents Break: A Field Analysis of Production Failures — Arize](https://arize.com/blog/common-ai-agent-failures/)
- Anthropic `claude-api` skill (bundled, first-party): `shared/evals/eval-audit.md` (eval health checklist, noise-floor formula, LLM-judge bias mitigations), `shared/cost-optimization.md` (Batch API 50% discount, prompt caching for repeated eval runs, cost-per-completed-task framing), `shared/error-codes.md` (rate limits, typed exception chains), `shared/tool-use-concepts.md` (parallel tool-call result batching, `max_tokens` truncation mid-tool-use, `pause_turn` handling) — current as of the skill's 2026-06-24 cache date
- `/Users/xuanhai/Desktop/cat/tracewell/.planning/PROJECT.md` and `/Users/xuanhai/Desktop/cat/tracewell/planning/PLAN.md` (project constraints and roadmap used to scope-match every pitfall above)

---
*Pitfalls research for: self-built agent tracing, eval harness, and checkpoint/recovery (TypeScript, portfolio project)*
*Researched: 2026-09-08*
