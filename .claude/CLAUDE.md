<!-- GSD:project-start source:PROJECT.md -->

## Project

**Tracewell — A Minimum Version of Braintrust**

A self-built, minimum version of Braintrust: an agent infrastructure layer covering tracing,
evaluation, observability, and checkpoint/recovery. The subject under observation is a small,
hand-written code-review agent built directly against the Claude API with no agent framework.
This is a portfolio and learning project for repositioning toward AI Engineering — not a product
to sell. Its audience is interviewers and the author.

**Core Value:** A live, screen-shareable demo of the full loop — a bug appears, the trace explains why, the eval
catches the regression — backed by an architecture teardown the author can defend in depth.

### Constraints

- **Tech stack**: TypeScript throughout — agent, tracing, eval, and dashboard — One runtime and one repo; avoids serialization overhead and context-switching between Python and TS
- **Tech stack**: Next.js + Supabase for the dashboard — Already familiar; the goal is to spend learning budget on the infra layer, not the UI layer
- **Tech stack**: No agent framework for the agent under test — The tool-calling loop is the thing being learned; a framework would hide it
- **Timeline**: Part-time over weeks — Phases must produce standalone value, since sessions are short and interruptible
- **Dependencies**: Claude API — The agent and the LLM-as-judge scorer both require it

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Build-vs-Adopt Summary

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

## 1. Tracing / Instrumentation

### Verdict: build a bespoke span schema; do not adopt the OpenTelemetry JS SDK as a runtime dependency

- **Build** your own span type (`{ id, traceId, parentSpanId, name, kind, startTime, endTime, attributes, events, status }`) and persist it directly to SQLite. This is Phase 1's actual deliverable — own it fully.
- **Borrow naming, not code.** Name your attributes after the `gen_ai.*` vocabulary where it fits your case exactly (`gen_ai.system` → `"anthropic"`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.operation.name`) so that if you ever want to bolt on OTel export later (a nice stretch goal for the teardown doc — "here's how this would plug into a real observability stack"), the field names already line up. This costs nothing today and is a legitimate thing to cite in the teardown as evidence of spec awareness.
- **Reference-only dependency, if any:** `@opentelemetry/semantic-conventions` (1.43.0) can be added as a documentation/type reference for attribute name constants (`ATTR_*`) without pulling in the SDK, tracer, or exporters — genuinely optional, skip it unless you want compile-time-checked attribute names.
- Do **not** reach for `@opentelemetry/sdk-node`, `@opentelemetry/sdk-trace-node` (2.11.0), or `@opentelemetry/exporter-trace-otlp-http` (0.222.0) for this project. They're excellent for what they are — production polyglot service tracing — and wrong-sized here.

## 2. Local Trace Persistence

### SQLite driver: `better-sqlite3` (adopt), not `node:sqlite` (not yet — reference only)

- Ships in Node since 22.5.0; runs without a flag since 22.13.0.
- On Node 24 (the environment here — confirmed `node --version` → v24.16.0, current Active LTS as of mid-2026) it sits at **Stability 1.1–1.2 ("Release Candidate")** — settled but not the final "Stable" stamp.
- Node 26 (released May 2026, enters LTS October 2026) is the first version to fully stabilize it.
- Functionally, it lags `better-sqlite3` in exactly the areas this project's hardest phase (Phase 3, checkpoint/recovery) needs most: `better-sqlite3` has a mature, well-tested synchronous transaction API (`db.transaction(fn)`), a first-class backup API, and support for custom SQL functions; `node:sqlite`'s surface is newer and thinner, with feature gaps still being found as adoption grows.

### Query layer: `drizzle-orm` (adopt), not raw SQL, not Kysely

- Drizzle has first-class, near-identical schema DSLs for both dialects: `sqliteTable(...)` and `pgTable(...)` use the same column-builder pattern (`integer()`, `text()`, `.primaryKey()`, `.notNull()`, `.references()`), so the SQLite→Postgres port at the dashboard phase is largely a matter of copying the schema file and swapping the table-helper import and the driver — not a rewrite.
- It has official, current drivers for all three SQLite backends discussed above (`drizzle-orm/better-sqlite3`, `drizzle-orm/node-sqlite`, `drizzle-orm/libsql`) and for Postgres via `postgres-js` (the documented path for Supabase — `drizzle({ client: postgres(process.env.DATABASE_URL!) })`).
- `drizzle-kit` (schema push/migrate) already normalizes SQLite file-path handling across the `libsql`/`better-sqlite3` drivers as of recent releases, reducing config friction.

## 3. Claude API Usage from TypeScript

### Tool-calling loop: build it yourself — do not use `toolRunner()`

### Streaming: adopt `client.messages.stream()`

### Cost/usage data: where it lives, and what you still have to build

- `input_tokens` (number)
- `output_tokens` (number)
- `cache_creation_input_tokens` (number, nullable)
- `cache_read_input_tokens` (number, nullable)
- `output_tokens_details.thinking_tokens` (number, when extended thinking is used)

## 4. Eval Harness Tooling

### Verdict: build the harness; do not adopt Evalite, promptfoo, or vitest-evals as the pipeline

| Tool | Version | What it is |
|---|---|---|
| `vitest` | 5.0.0 | General-purpose TS test runner (Vite-native) |
| `evalite` | 0.19.0 | Matt Pocock's Vitest-based eval framework — `.eval.ts` files, scoring, tracing, local UI, watch mode |
| `vitest-evals` | 0.16.1 | Sentry's Vitest extension — `describeEval()`, `toSatisfyJudge()`, built-in `FactualityJudge`/`ToolCallJudge`/`StructuredOutputJudge` |
| `promptfoo` | 0.122.2 | Standalone, YAML-config-driven, multi-language (JS/TS/Python) prompt-eval and red-teaming platform |

## 5. LLM-as-Judge Scoring

### Structured output: adopt the SDK's native `zodOutputFormat` + `messages.parse()`

### Schema validation: adopt `zod` v4 (4.5.4)

## 6. Dashboard

### Next.js + Supabase (already a project constraint) — how the migration lands in practice

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

# Core

# Dashboard (Phase 4 only)

# Dev dependencies

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

- Keep the bespoke span schema as the source of truth.
- Write a one-off exporter function that maps your span shape to `gen_ai.*`-named OTLP spans, using `@opentelemetry/api` purely to construct a `ReadableSpan`-shaped object for a demo export — not as the live tracer.
- Because your own attribute names already mirror `gen_ai.*` (per §1), this mapping is close to 1:1.
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

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
