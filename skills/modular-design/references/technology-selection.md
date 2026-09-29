# Technology Selection Reference

Choose technology **per capability**, behind a port, so the choice stays reversible. This reference gives the decision criteria first (framework-agnostic), then a worked modern stack as one concrete instantiation, then per-runtime notes.

The architecture does not depend on any row in these tables. That is the point: if swapping a row would force a domain change, the ACL is missing.

## Table of Contents

1. How to choose
2. Capability-by-capability criteria
3. A worked stack (TypeScript, 2026)
4. Equivalents in other ecosystems
5. Per-runtime notes
6. Key decisions and trade-offs
7. The evolution path

---

## 1. How to choose

For each capability, ask in this order:

1. **What does the domain need?** Express it as a port in domain language before naming any product.
2. **What is already in the codebase or the team's hands?** A slightly worse tool the team knows beats a better one they do not.
3. **What is the failure mode?** How does it degrade, and can the fallback be implemented?
4. **What is the exit cost?** If this turns out wrong in 18 months, what does replacing it touch? If the answer is "the whole domain", add an adapter first.
5. **Does it justify its operational cost?** Every extra piece of infrastructure is a thing to monitor, patch, secure, and page someone about at 3am.

Then **write the trade-off down** next to the choice. A technology table with no "why" column is a shopping list, not a decision record.

**Always ask the user before assuming.** Present alternatives with their trade-offs rather than silently picking.

---

## 2. Capability-by-capability criteria

| Capability | Choose on | Common options |
| --- | --- | --- |
| **Runtime** | Team familiarity · library ecosystem · concurrency model fit (I/O-bound vs CPU-bound) · long-running stability | Node/Bun/Deno · Python · Go · Java/Kotlin · .NET · Rust |
| **HTTP layer** | Throughput needs · middleware ecosystem · typing quality · structure the framework imposes | Fastify · Express · Koa · Hono · NestJS · FastAPI · Django · Rails · Spring · Gin/Echo · ASP.NET |
| **API style** | Who consumes it · whether external/non-TS clients exist · contract tooling | REST + OpenAPI (neutral, works for webhooks and external consumers) · GraphQL (client-shaped queries) · gRPC (internal, typed, fast) · tRPC (closed internal TS UI only) |
| **Persistence** | Consistency needs · query shape · migration story · operational maturity | Relational (PostgreSQL/MySQL) by default · document store when the shape is genuinely document-like · both, per module |
| **Data access** | Type safety · migration quality · escape hatch to raw SQL · N+1 protection | ORM (Prisma, TypeORM, SQLAlchemy, Hibernate, EF Core) · query builder (Kysely, Drizzle, jOOQ) · raw SQL + mapper |
| **Message bus** | Delivery guarantee · ordering · persistence · ops cost | DB-backed queue (start here) · managed queue (SQS/PubSub) · Redis/Valkey streams · Kafka (only when you have outgrown the rest) |
| **Cache** | Latency target · invalidation strategy · whether state must survive restarts | In-process (per instance) · Redis/Valkey (shared) · CDN (edge) |
| **Real-time** | Direction of traffic · proxy/infra constraints · client environment | SSE by default · WebSocket only for true bidirectional · managed push for mobile/serverless |
| **Validation** | Where the boundary is · schema reuse between transport and contract | Schema-first (Zod, Pydantic, JSON Schema) · decorator-based (class-validator) |
| **Observability** | Vendor neutrality · trace propagation across modules · cost at volume | OpenTelemetry (traces + metrics) + structured logging (Pino, structlog, zerolog) |
| **Auth** | Control needed · built-in features vs boilerplate · who owns the user tables | Token/session library with full control (Passport/JWT, Devise, Spring Security) vs a batteries-included provider (Better Auth, Auth0, Clerk, Cognito) |
| **Monorepo tooling** | Number of modules · CI time pressure · need for enforced boundaries | Nx (task graph, `affected`, enforced module boundaries) · Turborepo · Bazel · plain workspaces |
| **Lint/format** | Speed · one tool vs two · ecosystem rules | Biome (fast, single tool) · ESLint + Prettier (largest rule ecosystem) · Ruff · golangci-lint |
| **Testing** | Speed · framework integration · E2E needs | Vitest/Jest + Supertest/Playwright · pytest · go test · JUnit |

### Auth choice, in more detail

The decision is really "how much control do you need, and who owns the user tables?"

| Criterion | Roll-your-own (Passport/JWT, Spring Security) | Batteries-included (Better Auth, Auth0, Clerk) |
| --- | --- | --- |
| Maturity | Battle-tested, years in production | Newer or vendor-managed |
| Setup | More boilerplate, more control | Less boilerplate, convention-based |
| Social login | One strategy per provider | Built-in provider config |
| Sessions | Manual (token handling) | Built-in, often with cookie cache |
| 2FA / admin | Custom implementation | Plugin or feature flag |
| Custom logic | Full control at every step | Hooks and custom handlers |
| Database | You manage user/session tables | Auto-managed auth tables |

Either way: **authentication is a Generic subdomain.** It sits behind an `IdentityProviderPort`, and the rest of the system knows only "who is the caller and what may they do" — never the provider's session object.

---

## 3. A worked stack (TypeScript, 2026)

One concrete instantiation of the criteria above. Treat it as a reference point, not a requirement.

**Monorepo & runtime**

- **Nx** monorepo: `apps/` for composition roots (one deploy today; api/worker/scheduler later), `libs/` for bounded contexts + `shared/`. Enforce boundaries in CI with `enforce-module-boundaries`; use `nx affected` for incremental build/test. Tag projects by scope (`scope:billing`, `scope:shared`) and type (`type:domain-lib`), and allow only `scope:x → scope:shared`, never `scope:x → scope:y`.
- **Node 24 LTS** as the production runtime for the always-on API.
- **Bun** for dev, tests, and scripts (fast install, native TS). Keep Bun off the always-on critical path until validated in staging — its long-running GC behavior is less battle-tested than V8.
- **TypeScript** at the strictest settings, no `any`. **pnpm** as package manager.

**Non-negotiable compiler flags:** `strict`, `strictNullChecks`, `noImplicitAny`, `noUncheckedIndexedAccess`, plus `noImplicitReturns`, `noImplicitOverride`, `strictFunctionTypes`. These catch real bugs and enforce domain correctness. Path aliases map each module to its barrel (`@app/billing → libs/billing/src/index.ts`), which is what makes deep-import detection possible.

**Backend**

- **Fastify** (or NestJS on the Fastify adapter) — higher throughput than Express, good TS support, plugin architecture.
- **Prisma** ORM; **Zod** or class-validator for input validation; **OpenAPI** as the contract.
- Clean Architecture per module, organized flat-by-aggregate (`module-internals.md`).

**Frontend** (when there is one)

- **React 19** (Actions, `use`, `useOptimistic`, the React Compiler — drop manual `useMemo`/`useCallback`).
- **Vite** (native ESM), **Tailwind** (Rust engine), **TanStack Router** (type-safe, Zod search params), **TanStack Query** (Suspense, optimistic updates).
- **Orval** generates the typed client and query hooks from the backend OpenAPI — no hand-written fetch layer.
- **Vitest** + Testing Library; **Playwright** for E2E. **Biome** for lint/format. No `useEffect` for data fetching.

**Data & cache**

- **PostgreSQL** — one database; each module is the sole writer of its own tables (schema or prefix per context; no cross-context FKs).
- **Valkey** (the BSD fork of Redis OSS 7.2, under the Linux Foundation) for cache, rate limiting, circuit-breaker state, and SSE pub/sub. Drop-in for Redis; on managed cloud services it is roughly 20% cheaper node-based and up to ~33% serverless, with a zero-downtime upgrade path. Audit first if you depend on proprietary Redis modules (RediSearch, RedisJSON).

**Real-time, messaging, resilience**

- **SSE** for server-to-client push; Valkey pub/sub to fan out across instances.
- **Transactional outbox** for inter-module events; idempotent consumers.
- A resilience library (e.g. **Cockatiel**) for backoff + jitter, circuit breaker, and bulkhead.

**Observability & quality**

- **OpenTelemetry** (traces, metrics) + structured logging (**Pino**), with per-module context and correlation ids.
- `tsc --noEmit` in CI; coverage target ≥80%; Conventional Commits.

**Durable workflow**

- A durable-execution engine only for a long-running, must-resume lifecycle. If it is owned by another team, access it behind a `WorkflowOrchestrationPort` (ACL), and keep your own integrations (ERP, storage, notifications) as capabilities the workflow invokes — so you retain vendor-agnosticism.

---

## 4. Equivalents in other ecosystems

The architecture is unchanged; only the products differ.

| Capability | TypeScript | Python | Go | Java/Kotlin | .NET |
| --- | --- | --- | --- | --- | --- |
| HTTP | Fastify / NestJS | FastAPI / Django | Gin / Echo / chi | Spring Boot | ASP.NET Core |
| DI | Constructor injection or a container | Constructor injection / `dependency-injector` | Explicit constructor wiring | Spring | Built-in DI |
| Data access | Prisma / Kysely | SQLAlchemy | sqlc / GORM | Hibernate / jOOQ | EF Core / Dapper |
| Validation | Zod | Pydantic | validator + structs | Bean Validation | FluentValidation |
| Resilience | Cockatiel / opossum | tenacity + pybreaker | failsafe-go / gobreaker | resilience4j | Polly |
| Messaging | BullMQ / SQS SDK / KafkaJS | Celery / Kombu | Watermill / sarama | Spring Cloud Stream | MassTransit |
| Testing | Vitest / Jest | pytest | go test + testify | JUnit | xUnit |
| Tracing | OpenTelemetry JS | OpenTelemetry Python | OpenTelemetry Go | OpenTelemetry Java | OpenTelemetry .NET |

---

## 5. Per-runtime notes

**HTTP service.** The default shape. Composition root at bootstrap; controllers as transport adapters; the outbox relay as a background worker in the same process.

**Serverless.** Put expensive construction (connection pools, SDK clients, adapter graphs) at **module scope**, outside the handler, so warm invocations reuse it — but never keep mutable request state there. Use a connection pooler for relational databases (a per-invocation connection will exhaust the database). The outbox relay becomes a scheduled function polling the table, or a change-data-capture stream. Circuit-breaker state must live in a shared store to be accurate across containers. Cold-start cost makes heavy DI containers a poor fit — prefer explicit wiring.

**Queue / stream worker.** No HTTP surface, but still a public contract: the message schemas it consumes and publishes. Version them like an API. The broker's redelivery *is* the retry mechanism — configure visibility timeout and max receives instead of an in-process retry loop, and use a dead-letter queue as the terminal fallback. Consumers must be idempotent.

**Scheduled job.** Make it resumable and idempotent — the next scheduled run is the retry. Record progress so a crashed run does not redo completed work. Bound the total runtime.

**CLI / batch.** The argument parser is the transport adapter. Exit codes are the status codes. Make re-runs safe, and report partial progress on failure so the operator can resume rather than restart.

**Hybrid (most real systems).** One set of modules, several composition roots: `apps/api`, `apps/worker`, `apps/scheduler`, each wiring the same libraries with different transport adapters. This is exactly what P2 (Composability) and P7 (Deployment Independence) buy you — and why the module code must never assume which host it is running in.

---

## 6. Key decisions and trade-offs

- **Runtime:** a mature LTS runtime on the always-on service; faster experimental runtimes for dev, tests, and short-lived workers.
- **API style:** REST + OpenAPI + generated client by default — a neutral contract that also serves external consumers (webhooks) and non-TypeScript clients. Consider tRPC only for a closed internal UI with no external perimeter.
- **Real-time:** SSE by default; WebSocket only for true bidirectional needs.
- **Cache:** Valkey over Redis for license and cost, unless you need proprietary Redis modules.
- **Messaging:** the simplest transport that meets the delivery guarantee. A DB-backed queue with the outbox you already have beats introducing Kafka on day one.
- **CQRS / Event Sourcing:** not by default. Only when the read/write divergence or the audit requirement is real.
- **Granularity:** one deploy now; promote a module to its own app or database only when *its own* metrics justify it. Evolution, not big-bang.

---

## 7. The evolution path

State it explicitly in every design document.

**Stage 1 — Modular monolith (start here).** One deployable unit, one database. Strong logical boundaries: modules own their state, communicate through facades and events, and every external system sits behind an ACL. The outbox and idempotent consumers are already in place — this is what makes the later stages cheap.

**Stage 2 — Split the runtime, keep the data.** Add a worker app and/or a scheduler app that compose the *same* module libraries with different transport adapters. No contract changes; the relay moves out of the API process. Trigger: background work competes with request latency, or the two need different scaling.

**Stage 3 — Promote one module to its own deployable.** The events already flow over a real transport, so the in-process facade call becomes a remote call behind the same port. Trigger: that module's own metrics — its scaling profile, its failure blast radius, its release cadence, or its team ownership — justify the operational cost.

**Stage 4 — Give that module its own database.** Only after step 3 is stable, and only for a module whose state is genuinely self-contained (which the no-cross-context-FK rule has already guaranteed).

**Never reverse-engineer microservices from hype.** Each stage must be justified by a measured problem in the stage before it. Record the trigger condition for the next stage so the team knows what to watch for.
