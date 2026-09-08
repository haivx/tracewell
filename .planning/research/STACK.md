# Stack Research

**Domain:** TypeScript agent-observability and evaluation infrastructure (self-built minimum Braintrust — tracing, eval harness, checkpoint/recovery, dashboard)
**Researched:** 2026-09-08
**Confidence:** HIGH (all versions verified live via Context7/npm registry on 2026-09-08; framework-adoption judgment calls are MEDIUM — they depend on the project's stated learning goals, not on external fact)

## Build-vs-Adopt Summary

This is the single table the roadmap should read first. Everything below justifies it.

| Component | Decision | Confidence |
|---|---|---|
| Trace span schema & tracer | **BUILD** — hand-roll a bespoke schema, informed by OTel `gen_ai.*` naming | HIGH |
| OpenTelemetry JS SDK (full runtime dependency) | **DO NOT ADOPT** — reference the spec, don't run the SDK | HIGH |
| SQLite driver | **ADOPT** `better-sqlite3` | HIGH |
| Query layer | **ADOPT** `drizzle-orm` + `drizzle-kit` | HIGH |
| Claude API client | **ADOPT** `@anthropic-ai/sdk` (the HTTP/streaming/typing layer) | HIGH |
| Tool-calling loop (`toolRunner` helper) | **BUILD** — write the manual `while` loop yourself; do not use `client.beta.messages.toolRunner()` | HIGH (this is an explicit project constraint, not just a preference) |
| Streaming accumulation (`client.messages.stream()`) | **ADOPT** — pure SSE-parsing boilerplate, no learning lost | HIGH |
| Cost/usage extraction | **BUILD** the cost math (pricing table + arithmetic); **ADOPT** the SDK's `usage` fields as the data source | HIGH |
| Eval harness / runner (`eval run` command, comparison table, score-drop gate) | **BUILD** — plain TypeScript CLI, not Evalite/promptfoo/vitest-evals | HIGH |
| Vitest (as the project's ordinary unit-test runner) | **ADOPT** — for testing the codebase itself, not as the eval pipeline | HIGH |
| Deterministic scorers (string/edit-distance/etc.) | **BUILD** (scope is 10–20 cases; hand-rolling costs minutes) — `autoevals` is an optional narrow adopt if time-constrained | MEDIUM |
| LLM-as-judge structured output | **ADOPT** `zodOutputFormat` + `client.messages.parse()`; **BUILD** the rubric/prompt design | HIGH |
| Schema validation | **ADOPT** `zod` v4 | HIGH |
| Dashboard framework | **ADOPT** `Next.js` (already a project constraint) | HIGH |
| Dashboard data layer | **ADOPT** `@supabase/supabase-js` + Postgres via Drizzle | HIGH |

---

## 1. Tracing / Instrumentation

### Verdict: build a bespoke span schema; do not adopt the OpenTelemetry JS SDK as a runtime dependency

**What the OTel GenAI semantic conventions actually are right now (verified 2026-09-08):** every `gen_ai.*` attribute, span, metric, and event in the official registry still carries the stability badge **"Development"** (OTel's renamed "experimental" tier) — none has graduated to Stable. As of the v1.42.0 release (June 12, 2026), the GenAI conventions were split out of the core `opentelemetry-specification` repo into their own dedicated repo, which is an organizational move to let GenAI iterate faster — explicitly **not** a stabilization signal. There is no public timeline for when `gen_ai.*` will stabilize, and attribute names/shapes can still change.

**Why this matters for a solo, weeks-long project:** adopting the full OTel JS pipeline (`@opentelemetry/sdk-node` 0.222.0, a `TracerProvider`, span processors, an OTLP exporter, a context propagator) buys you interoperability with tools like Jaeger/Honeycomb/Grafana Tempo — value this project has no use for, since the deliverable is a self-hosted SQLite/Postgres trace store and a custom Next.js waterfall viewer, not a vendor backend. What it costs you is a layer of indirection (context managers, span processors, resource detectors) sitting directly on top of the one thing the project exists to teach: **the trace schema itself** — spans, parent-child nesting, latency, cost. Running that logic through someone else's SDK is exactly the kind of "library hides the mechanism" the project explicitly wants to avoid.

There is also no official Anthropic auto-instrumentation in `opentelemetry-js-contrib` — the only Claude-specific OTel instrumentations are third-party (`@traceloop/instrumentation-anthropic`, `@arizeai/openinference-instrumentation-anthropic`), each with its own attribute conventions layered on top of the unstable base spec. Adopting one of those would mean depending on a community wrapper around a spec that itself hasn't settled.

**Recommendation:**
- **Build** your own span type (`{ id, traceId, parentSpanId, name, kind, startTime, endTime, attributes, events, status }`) and persist it directly to SQLite. This is Phase 1's actual deliverable — own it fully.
- **Borrow naming, not code.** Name your attributes after the `gen_ai.*` vocabulary where it fits your case exactly (`gen_ai.system` → `"anthropic"`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.operation.name`) so that if you ever want to bolt on OTel export later (a nice stretch goal for the teardown doc — "here's how this would plug into a real observability stack"), the field names already line up. This costs nothing today and is a legitimate thing to cite in the teardown as evidence of spec awareness.
- **Reference-only dependency, if any:** `@opentelemetry/semantic-conventions` (1.43.0) can be added as a documentation/type reference for attribute name constants (`ATTR_*`) without pulling in the SDK, tracer, or exporters — genuinely optional, skip it unless you want compile-time-checked attribute names.
- Do **not** reach for `@opentelemetry/sdk-node`, `@opentelemetry/sdk-trace-node` (2.11.0), or `@opentelemetry/exporter-trace-otlp-http` (0.222.0) for this project. They're excellent for what they are — production polyglot service tracing — and wrong-sized here.

**Confidence:** HIGH on the spec-stability facts (verified against WebSearch results dated May–July 2026 and the v1.42.0 release note). MEDIUM-to-HIGH on the build-vs-adopt call — it follows directly from the project's own stated constraint ("where a library would hide a mechanism the author explicitly wants to understand... say so and recommend building it"), which is about as unambiguous as these calls get.

---

## 2. Local Trace Persistence

### SQLite driver: `better-sqlite3` (adopt), not `node:sqlite` (not yet — reference only)

Both were seriously considered since `node:sqlite` is the more novel, zero-dependency option and would be an interesting thing to cite in a 2026 teardown. Current status of `node:sqlite`, verified 2026-09-08:

- Ships in Node since 22.5.0; runs without a flag since 22.13.0.
- On Node 24 (the environment here — confirmed `node --version` → v24.16.0, current Active LTS as of mid-2026) it sits at **Stability 1.1–1.2 ("Release Candidate")** — settled but not the final "Stable" stamp.
- Node 26 (released May 2026, enters LTS October 2026) is the first version to fully stabilize it.
- Functionally, it lags `better-sqlite3` in exactly the areas this project's hardest phase (Phase 3, checkpoint/recovery) needs most: `better-sqlite3` has a mature, well-tested synchronous transaction API (`db.transaction(fn)`), a first-class backup API, and support for custom SQL functions; `node:sqlite`'s surface is newer and thinner, with feature gaps still being found as adoption grows.

Given that checkpoint/recovery is explicitly "the hardest and most interesting" phase and depends on transactional durability, this is not the place to also be absorbing risk from an RC-stability runtime module. **`better-sqlite3` (13.0.3)** is the pick: fully synchronous API (fits a single-process CLI agent — no async ceremony for what is fundamentally sequential work), requires Node ≥22 (satisfied), ships prebuilt binaries for common platforms (no compiler toolchain needed in the typical case), and is the most widely supported driver across the query-layer ecosystem (Drizzle and Kysely both treat it as the reference SQLite driver).

Enable WAL mode (`db.pragma('journal_mode = WAL')`) immediately — it's a one-liner, meaningfully improves concurrent read/write performance, and matters once the Next.js dashboard (Phase 4) is reading the same SQLite file a CLI run is writing to.

**Do not use** `libsql`/`@libsql/client` (0.18.0) as the primary local driver. libSQL (Turso's SQLite fork) remains actively maintained and is not going away, but as of 2026 Turso's own product focus has shifted to a new, from-scratch Rust database ("Turso Database") — libSQL's headline value (embedded replicas, edge sync to a remote Turso Cloud database) solves a distributed-systems problem this project doesn't have. Reach for it only if a later stretch goal wants a synced remote replica before the Postgres migration; it isn't a better plain-local-file SQLite driver than `better-sqlite3`.

### Query layer: `drizzle-orm` (adopt), not raw SQL, not Kysely

**Adopt Drizzle** (`drizzle-orm` 0.45.2, `drizzle-kit` 0.31.10). This is pure infrastructure plumbing — parameterized-query construction and TypeScript typing — not a mechanism the project is trying to teach, so there's no learning cost to adopting it, and it directly derisks the one migration the project has explicitly planned:

- Drizzle has first-class, near-identical schema DSLs for both dialects: `sqliteTable(...)` and `pgTable(...)` use the same column-builder pattern (`integer()`, `text()`, `.primaryKey()`, `.notNull()`, `.references()`), so the SQLite→Postgres port at the dashboard phase is largely a matter of copying the schema file and swapping the table-helper import and the driver — not a rewrite.
- It has official, current drivers for all three SQLite backends discussed above (`drizzle-orm/better-sqlite3`, `drizzle-orm/node-sqlite`, `drizzle-orm/libsql`) and for Postgres via `postgres-js` (the documented path for Supabase — `drizzle({ client: postgres(process.env.DATABASE_URL!) })`).
- `drizzle-kit` (schema push/migrate) already normalizes SQLite file-path handling across the `libsql`/`better-sqlite3` drivers as of recent releases, reducing config friction.

**Kysely** (0.29.5) was the other serious candidate and remains a legitimate, high-quality alternative — it's a thinner, more "SQL-shaped" type-safe query builder with excellent Postgres/SQLite support. The reason to prefer Drizzle here specifically is the schema-DSL parity across dialects: Kysely's typing is derived from a `Database` interface you write by hand per-dialect (or generate via `kysely-codegen`), so the SQLite→Postgres move means re-deriving/adjusting that interface rather than swapping one import. For a project whose whole reason to start on SQLite is "pay the one migration later," Drizzle's schema-file-first design is the better fit.

**Migration pattern for this shape of data (SQLite → Supabase/Postgres):** keep one `schema.ts` file using Drizzle's table builders; when the dashboard phase arrives, add a second `schema.pg.ts` alongside it (or swap the import at that phase, since Phases 1–3 are done and only the dashboard reads the old file), run `drizzle-kit generate` against the Postgres schema, and use the Supabase CLI (`supabase db push` / `supabase migration up`) to apply it to the hosted project. Data migration itself (copying rows) is a one-off script: read all rows via the SQLite Drizzle client, batch-insert via the Postgres Drizzle client — trivial at this project's scale (dozens of runs, not millions of spans), so no ETL tooling is warranted.

---

## 3. Claude API Usage from TypeScript

**Package:** `@anthropic-ai/sdk` **0.124.0** (verified current on npm 2026-09-08; Context7 library ID `/anthropics/anthropic-sdk-typescript`).

### Tool-calling loop: build it yourself — do not use `toolRunner()`

The SDK ships a beta helper, `client.beta.messages.toolRunner()` (paired with `betaZodTool()` for Zod-typed tool definitions), that fully automates the request→execute→loop cycle: you hand it tools and messages, it drives everything until the model stops calling tools, exposing `await runner` for the final message or `for await` for intermediate turns.

This is a good helper in general — but this project has an explicit, first-class constraint that it exists specifically to avoid handing the tool-calling loop to any library, framework or SDK helper: understanding `stop_reason === "tool_use"` handling, `tool_use`/`tool_result` content-block shapes, and parallel-tool-call semantics at the primitive level is Phase 0's whole point. Using `toolRunner()` would satisfy the letter of "no agent framework" while quietly reintroducing exactly the abstraction the constraint is written to prevent.

**Build:** a manual loop against `client.messages.create()` (or `.stream()` — see below): send messages, check `response.stop_reason`, iterate `response.content` for `tool_use` blocks, execute your tool handlers, and return **all** resulting `tool_result` blocks in a single subsequent user message (the API trains on parallel tool-call batches — splitting them across messages silently discourages the model from batching calls). This is maybe 40–60 lines of code, all of it directly the mechanism the project cares about.

### Streaming: adopt `client.messages.stream()`

For interactive CLI output, use the SDK's streaming helper rather than the manual `create({ stream: true })` async iterable. `client.messages.stream()` returns a `MessageStream` that accumulates SSE deltas into a structured message for you and exposes `.finalMessage()` / `.finalText()` plus a rich event surface (`'text'`, `'contentBlock'`, `'message'`). This is pure SSE-chunk-accumulation bookkeeping — there is no domain insight lost by not hand-rolling it, and reimplementing an event accumulator is exactly the kind of "reimplementing SDK plumbing" the SDK's own guidance warns against. Adopt it outright.

### Cost/usage data: where it lives, and what you still have to build

Every `Message` response carries a `usage` object with:

- `input_tokens` (number)
- `output_tokens` (number)
- `cache_creation_input_tokens` (number, nullable)
- `cache_read_input_tokens` (number, nullable)
- `output_tokens_details.thinking_tokens` (number, when extended thinking is used)

Per the SDK's own type documentation: **total input tokens = `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`.** During streaming, the delta-accumulation logic in the SDK treats `output_tokens` as always-overwrite (it's a running total) but the three input-side counters as "set when present, never summed" — worth knowing if you ever accumulate usage across a stream yourself rather than reading it off the final message.

There is no dollar-cost field anywhere in the API — **you build the cost math**: a small static pricing table (`$/MTok` input and output, per model) joined against the `usage` object per span, summed per trace. Current relevant prices (verified 2026-09-08): Claude Sonnet 5 (`claude-sonnet-5`) — $2.00 / $10.00 per MTok in/out; Claude Haiku 4.5 (`claude-haiku-4-5`) — $1.00 / $5.00 per MTok in/out. A sensible default: run the code-review agent under test on Sonnet 5, and run the LLM-as-judge scorer on Haiku 4.5 to keep eval-run costs low (bump to Sonnet 5 for the judge only if judgment quality on the demo case turns out to need it — cheap to A/B, since it's one model-id swap).

**Confidence:** HIGH — all of the above is verified directly against SDK source/type comments via Context7, not recalled from training data.

---

## 4. Eval Harness Tooling

### Verdict: build the harness; do not adopt Evalite, promptfoo, or vitest-evals as the pipeline

Four real options exist in the TS ecosystem as of 2026, all verified live:

| Tool | Version | What it is |
|---|---|---|
| `vitest` | 5.0.0 | General-purpose TS test runner (Vite-native) |
| `evalite` | 0.19.0 | Matt Pocock's Vitest-based eval framework — `.eval.ts` files, scoring, tracing, local UI, watch mode |
| `vitest-evals` | 0.16.1 | Sentry's Vitest extension — `describeEval()`, `toSatisfyJudge()`, built-in `FactualityJudge`/`ToolCallJudge`/`StructuredOutputJudge` |
| `promptfoo` | 0.122.2 | Standalone, YAML-config-driven, multi-language (JS/TS/Python) prompt-eval and red-teaming platform |

Evalite and vitest-evals are, on their technical merits, well-built and specifically aimed at exactly this problem — they are not agent frameworks, and the project's constraint about avoiding those doesn't disqualify them by rule. They're excluded on a different, more direct ground: the project's own roadmap names **"an eval harness... combining hard assertions with LLM-as-judge... comparison of two named runs... an `eval run` command producing a comparison table"** as Phase 2's deliverable, and separately calls out checkpoint/recovery and "the eval quality gate" as deliberately split into their own phase specifically because that's "the hardest and most interesting work." That is a direct statement that eval-as-infra — the run harness, the scoring pipeline, the cross-run comparison, the score-drop gate — **is the subject matter**, not incidental tooling around it. Evalite and vitest-evals would hand you a working version of precisely that pipeline (run storage, comparison, judge orchestration) out of the box. Adopting either produces the deliverable without producing the understanding — the same failure mode the project's "no agent framework" rule is written to prevent, just one layer up the stack.

`promptfoo` is excluded for an additional, more mechanical reason: it's a standalone product with its own YAML-driven run format and its own CLI — running your agent through it means the agent executes as a black box inside promptfoo's process, not inside your own tracer-wrapped harness. Since every eval run also needs to emit spans into the *same* trace store the CLI agent writes to (that integration is itself part of the demo — "the eval catches the regression" needs to be traceable), routing eval execution through a separate tool's process model works against that goal.

**Build:** a plain TypeScript CLI script (run via `tsx` or compiled with `tsc`) that: loads your 10–20 fixture cases, runs the agent-under-test through your own traced harness for each, applies hard assertions (lint/test pass-fail — plain functions, no framework needed) plus an LLM-as-judge scorer (see §5), stores per-case and per-run results (in the same SQLite database as traces — a `runs`/`eval_results` table), and prints/exports a comparison table between two named runs. This is squarely inside the "weeks, part-time, standalone-value-per-phase" shape the project is scoped for.

**Where Vitest still belongs:** as the project's ordinary unit-test runner for the codebase itself (tracer logic, checkpoint/recovery logic, agent tool handlers). That's conventional software testing, not the eval-as-infra mechanism — no learning is forfeited by using a standard runner for it, and it's already implied by "TypeScript throughout." Just don't route the *eval* pipeline through it via `vitest-evals`; keep the eval CLI a separate, purpose-built script.

**`autoevals`** (Braintrust's scorer library, 0.3.0) is worth a one-line mention as an optional, narrow adopt: it provides deterministic scorers (Levenshtein/edit-distance, embedding similarity, etc.) as plain functions usable outside the full Braintrust SDK. At 10–20 cases, hand-writing the two or three deterministic checks you actually need is a matter of minutes and keeps the whole scorer surface visible in your own code for the teardown — so the default recommendation is still to skip it, but if a specific case genuinely needs (say) fuzzy string matching, pulling in just that one scorer function is a reasonable, low-risk exception. It is not an agent framework and not disqualified by that rule; it's excluded purely on "not enough time saved to justify the dependency" grounds at this scope.

**Confidence:** HIGH on what each tool is (verified via GitHub source + npm). MEDIUM-HIGH on the build-vs-adopt call for Evalite/vitest-evals — it follows from the project's roadmap language, which is about as close to an explicit instruction as research can get, but is ultimately a judgment call about *how much* of the pipeline counts as "the thing being learned."

---

## 5. LLM-as-Judge Scoring

### Structured output: adopt the SDK's native `zodOutputFormat` + `messages.parse()`

`@anthropic-ai/sdk` ships first-party, non-beta-namespaced structured-output support (the `structured-outputs-2025-12-15` capability is auto-injected as a beta header internally by `.parse()` — no manual header wrangling needed):

```typescript
import { zodOutputFormat } from '@anthropic-ai/sdk/helpers/zod';
import { z } from 'zod';

const JudgeVerdict = z.object({
  score: z.number().min(0).max(1),
  verdict: z.enum(['pass', 'fail']),
  reasoning: z.string(),
});

const message = await client.messages.parse({
  model: 'claude-haiku-4-5',
  max_tokens: 1024,
  messages: [{ role: 'user', content: judgePrompt }],
  output_config: { format: zodOutputFormat(JudgeVerdict) },
});

message.parsed_output; // typed as { score: number; verdict: 'pass'|'fail'; reasoning: string }
```

This constrains the model's output at generation time *and* validates/parses it client-side against the same schema in one call — this is exactly the "schema-validated model output" the research question asks about, and it's built into the SDK you're already using, not a separate library. Adopt it outright; there's no learning value in hand-rolling JSON-schema-constrained decoding or a bespoke retry-on-invalid-JSON loop when the SDK already does both correctly. (A raw-JSON-Schema variant, `jsonSchemaOutputFormat`, exists too, but Zod is the better fit here since you're already using Zod for the rest of your schema validation — see below.)

### Schema validation: adopt `zod` v4 (4.5.4)

Zod is the de facto standard for TS-first schema validation and static type inference, and as of v4 it ships a native `z.toJSONSchema(schema)` converter (Draft 2020-12 by default, with `draft-07`/`draft-04`/`openapi-3.0` targets available) — useful if you ever need to hand a raw JSON Schema to something other than the Anthropic SDK's own `zodOutputFormat` helper (which already does this conversion for you internally). Use Zod schemas for: the judge's output shape (above), your trace-span/event schema on the way into SQLite, and your eval-case fixture format. One validation library across the whole codebase is the right call for a solo project — there's no case here for a second, competing validation library (e.g., `io-ts`, `valibot`, `typebox`); Zod's ecosystem integration (Drizzle, the Anthropic SDK helper, most tooling) is the deciding factor.

**Build, deliberately:** the rubric/prompt design for the judge itself — what the judge is told to check, how it's told to weigh partial credit, how its reasoning is surfaced in the eval report. That's the actual interesting design problem in "LLM-as-judge," and no library can do it for you; `autoevals`' `Factuality` scorer is a reasonable reference to read for prompt-design ideas, but writing your own keeps the rubric visible and tunable, which matters for the demo ("the eval catches the regression" needs a rubric the author can explain and defend).

**Confidence:** HIGH — `zodOutputFormat`/`messages.parse()` and `z.toJSONSchema()` both verified directly against source docs via Context7.

---

## 6. Dashboard

### Next.js + Supabase (already a project constraint) — how the migration lands in practice

**Next.js 16.3.4** (current stable; verified via npm dist-tags, `latest` = 16.3.4, requires Node ≥20.9.0 — satisfied by Node 24) and **`@supabase/supabase-js` 2.116.0** are both project-given constraints, not open research questions — the brief is explicit that dashboard-layer learning budget is deliberately not being spent here. The only real technical question is how the SQLite→Postgres handoff should work, which is covered in §2's migration pattern: Drizzle schema ported (not rewritten) to `pgTable`, `drizzle-kit generate` against the Postgres schema, `supabase db push`/`migration up` to apply it, and a one-off row-copy script for the (small) existing trace/eval data. Because trace and eval-result data for this project is bounded (a handful of demo runs, not a production volume), there's no need for streaming ETL, batching frameworks, or a dual-write period — a synchronous copy script run once at the start of Phase 4 is sufficient.

Use Drizzle (not the Supabase JS client's query builder) for the dashboard's actual data access, and `@supabase/supabase-js` only for what Drizzle-over-Postgres doesn't cover if it comes up (e.g., Realtime subscriptions, if a "live trace as it streams in" UI touch is wanted for the demo — optional, not required for the core deliverable).

**Confidence:** HIGH — versions verified; the migration pattern is a direct extension of §2's Drizzle rationale, not new research.

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `@anthropic-ai/sdk` | 0.124.0 | Claude API client — messages, streaming, structured outputs | Official SDK; typed request/response shapes; `usage` fields are the cost-per-span data source |
| `better-sqlite3` | 13.0.3 | Local trace/eval store engine | Mature, synchronous, first-class transactions/backups — matters for checkpoint/recovery; requires Node ≥22 (have Node 24) |
| `drizzle-orm` + `drizzle-kit` | 0.45.2 / 0.31.10 | Type-safe query layer over SQLite now, Postgres later | Near-identical schema DSL across both dialects — directly derisks the planned SQLite→Supabase migration |
| `zod` | 4.5.4 | Schema validation (judge output, trace schema, fixtures) | TS-first, native `toJSONSchema()`, first-class integration with the Anthropic SDK's `zodOutputFormat` helper |
| `next` | 16.3.4 | Dashboard framework (Phase 4) | Project constraint; requires Node ≥20.9.0 |
| `@supabase/supabase-js` | 2.116.0 | Postgres/Realtime client for the dashboard | Project constraint; used alongside Drizzle-over-Postgres |
| `vitest` | 5.0.0 | Unit-test runner for the codebase itself | Standard TS test runner; NOT used as the eval pipeline (see §4) |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `tsx` | (latest) | Run TS files directly (agent CLI, eval CLI) without a build step | Any local script execution during Phases 0–3 |
| `@opentelemetry/semantic-conventions` | 1.43.0 | Reference-only: `ATTR_*` constant names for `gen_ai.*` | Optional — only if you want compile-time-checked attribute names when naming your own span fields; do not pull in the SDK/tracer alongside it |
| `autoevals` | 0.3.0 | Deterministic scorer functions (Levenshtein, etc.) | Optional, narrow use only if a specific eval case needs a scorer not worth hand-writing (see §4) |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `drizzle-kit` | Schema migrations/push for both SQLite and Postgres | `drizzle-kit generate` / `drizzle-kit push`; normalizes SQLite file URLs across drivers |
| Supabase CLI | Apply migrations to the hosted Postgres project | `supabase db push`, `supabase migration up` — used once, at the Phase 4 transition |

## Installation

```bash
# Core
npm install @anthropic-ai/sdk better-sqlite3 drizzle-orm zod

# Dashboard (Phase 4 only)
npm install next @supabase/supabase-js postgres

# Dev dependencies
npm install -D drizzle-kit vitest tsx typescript @types/better-sqlite3
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|--------------------------|
| `better-sqlite3` | `node:sqlite` (built-in) | If you specifically want to demonstrate/discuss the newest Node built-in in the teardown, and are comfortable with RC-stability risk and its thinner transaction/backup surface; not recommended for the checkpoint/recovery phase specifically |
| `better-sqlite3` | `@libsql/client` (libSQL/Turso) | If a later stretch goal wants a synced remote replica ahead of the Postgres migration; adds no value for a purely local single-writer store |
| `drizzle-orm` | `kysely` | If you prefer a thinner, more SQL-shaped query builder and don't mind re-deriving your `Database` type interface by hand when porting SQLite→Postgres |
| Bespoke trace schema | Full OpenTelemetry JS SDK + OTLP export | If the real goal shifts to interop with an existing observability backend (Honeycomb, Grafana Tempo) rather than a self-built store/dashboard — not this project's goal |
| Hand-rolled eval CLI | `evalite` | If the project's goal were "get a working eval pipeline fast" rather than "understand eval-as-infra" — a legitimate choice for a different kind of project, not this one |
| Hand-rolled eval CLI | `vitest-evals` | Same as above; also a reasonable pattern reference to read even if not adopted |
| Zod judge rubric | `autoevals` `Factuality` scorer | If time-constrained and willing to trade rubric transparency for speed |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|--------------|
| `client.beta.messages.toolRunner()` | Automates exactly the tool-calling loop this project exists to teach | Hand-write the `while (stop_reason === 'tool_use')` loop |
| Full OTel JS SDK (`@opentelemetry/sdk-node`, exporters, `TracerProvider`) | `gen_ai.*` conventions are still "Development" stability with no stabilization timeline; the SDK layer adds indirection over the schema you're meant to design yourself | Bespoke span type persisted directly to SQLite; borrow `gen_ai.*` naming only |
| `evalite` / `vitest-evals` as the eval pipeline | Would deliver Phase 2's stated learning objective (the harness, the comparison table, the score gate) pre-built | Hand-written eval CLI script |
| `promptfoo` | Runs your agent as a black box inside its own process/config format; breaks the "every eval run also emits spans into the same trace store" integration | Hand-written eval CLI script calling your own traced harness |
| `node:sqlite` for this project (for now) | RC stability on the current Node LTS; thinner transaction/backup/custom-function surface than `better-sqlite3`, which matters for the checkpoint/recovery phase | `better-sqlite3` |
| `@libsql/client` as the primary local driver | Solves a distributed-replica problem this project doesn't have; Turso's product focus has moved on from libSQL as the flagship | `better-sqlite3` locally, Postgres via Drizzle at the dashboard phase |

## Stack Patterns by Variant

**If a later phase wants OTel-compatible export for the teardown's "how this would scale" section:**
- Keep the bespoke span schema as the source of truth.
- Write a one-off exporter function that maps your span shape to `gen_ai.*`-named OTLP spans, using `@opentelemetry/api` purely to construct a `ReadableSpan`-shaped object for a demo export — not as the live tracer.
- Because your own attribute names already mirror `gen_ai.*` (per §1), this mapping is close to 1:1.

**If time pressure forces a scope cut on the eval harness:**
- Keep the hand-rolled CLI structure (fixture loop → traced run → assertions → judge → comparison table) — that's the load-bearing learning artifact.
- It's fine to lean on `autoevals` for one or two deterministic scorers rather than hand-writing every check, without compromising the harness itself.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|------------------|-------|
| `better-sqlite3@13.0.3` | Node ≥22 | Node 24 (current environment) satisfies this |
| `drizzle-orm@0.45.2` | `better-sqlite3`, `node:sqlite`, `@libsql/client`, `postgres` (postgres-js) | Same schema-builder pattern across SQLite and Postgres dialects; different table-helper import (`sqliteTable` vs `pgTable`) |
| `next@16.3.4` | Node ≥20.9.0 | Node 24 satisfies this |
| `@anthropic-ai/sdk@0.124.0` | `zod` (any v4.x) via `@anthropic-ai/sdk/helpers/zod` | `zodOutputFormat` auto-injects the `structured-outputs-2025-12-15` capability — no manual beta header needed |

## Sources

- `/anthropics/anthropic-sdk-typescript` (Context7) — tool use / `toolRunner`, streaming (`messages.stream`), `usage` field semantics, `zodOutputFormat`/`messages.parse`
- `/wiselibs/better-sqlite3` (Context7) — API basics, WAL mode, durability tradeoffs
- `/drizzle-team/drizzle-orm-docs` (Context7) — SQLite schema DSL, `node-sqlite` driver support, Supabase/Postgres connection and migration flow
- `/open-telemetry/opentelemetry-js` (Context7) — NodeSDK setup, span/context API shape (used to confirm the "adopt or not" cost, not to justify adoption)
- `/colinhacks/zod` (Context7) — `z.toJSONSchema()`
- npm registry (`npm view`, queried 2026-09-08) — exact current versions for all packages listed above
- [OpenTelemetry's GenAI semantic conventions are NOT stable yet — dev.to, 2026](https://dev.to/azena-ai/opentelemetrys-genai-semantic-conventions-are-not-stable-yet-heres-what-actually-shipped-in-2026-3mke) — `gen_ai.*` stability status, repo split
- [The state of the OpenTelemetry GenAI semantic conventions (July 2026) — John Hodge](https://john-hodge.com/blog/opentelemetry-genai-semantic-conventions/) — corroborating stability status
- [Node.js — Evolving the Node.js Release Schedule](https://nodejs.org/en/blog/announcements/evolving-the-nodejs-release-schedule) / [Node.js 26 vs Node.js 24 LTS](https://gethired.dev/blog/nodejs-26-vs-nodejs-24-lts-upgrade/) — Node 24 Active LTS / Node 26 stabilizes `node:sqlite`
- [`node:sqlite` and benchmarking — WiseLibs/better-sqlite3#1266](https://github.com/WiseLibs/better-sqlite3/issues/1266) / [future of better-sqlite3 vs node:sqlite — #1234](https://github.com/WiseLibs/better-sqlite3/issues/1234) — feature/maturity gap discussion
- [Pekka Enberg (Turso) on X — Turso/Turso Cloud/libSQL relationship](https://x.com/penberg/status/2032373944007688226) — libSQL maintenance status vs Turso's product focus
- [mattpocock/evalite (GitHub)](https://github.com/mattpocock/evalite) — Evalite scope and maturity
- [getsentry/vitest-evals (GitHub)](https://github.com/getsentry/vitest-evals) — vitest-evals scope, judges, maturity
- [Anthropic Claude API reference (claude-api skill, cached 2026-06-24, cross-checked against `client.models.list()` semantics)](https://docs.claude.com) — current model IDs and pricing (`claude-sonnet-5`, `claude-haiku-4-5`)

---
*Stack research for: TypeScript agent-observability and evaluation infrastructure*
*Researched: 2026-09-08*
