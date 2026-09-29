---
name: modular-design
description: Framework-agnostic guidance for designing, building, and reviewing modular systems — bounded contexts, module boundaries, state ownership, explicit contracts, ports and adapters (Anti-Corruption Layer), event-driven communication with a transactional outbox, resilience, coupling analysis, and monolith decomposition. Applies to any runtime — HTTP servers (Express, Fastify, Koa, NestJS, Django, Rails, Spring), serverless functions, queue workers, cron jobs, CLIs, and background daemons. Use when designing a new backend or platform, defining bounded contexts and a context map, laying out modules and folders, deciding monolith vs services, decoupling from a vendor, making calls resilient, measuring coupling, splitting a monolith, or reviewing module boundaries. Triggers on "modular monolith", "bounded context", "module boundaries", "ports and adapters", "anti-corruption layer", "how should we split this", "coupling analysis", "state isolation", "domain grouping", "decomposition", "screaming architecture". Do NOT use for simple CRUD with a handful of endpoints, framework-specific API syntax, UI component architecture, or infrastructure capacity planning.
license: CC-BY-4.0
metadata:
  version: 1.0.0
  derived-from: evolutionary-modular-architecture, nestjs-modular-monolith, modular-design-principles, modular-decomposition
  original-author: Felipe Rodrigues - github.com/felipfr (portions, CC-BY-4.0)
---

# Modular Design

Design, build, and review systems as **modules with strong logical boundaries** — regardless of language, framework, or runtime shape. A bounded context, a public contract, a single writer per piece of state, and an adapter around every vendor are the same ideas whether the code runs behind Express, inside a Lambda, in a Kafka consumer, or in a CLI.

This skill is the single compiled source for: modular + structural principles, DDD strategic and tactical design, module internals (flat-by-aggregate), contracts and communication, state isolation, resilience, coupling analysis, and monolith decomposition.

## When to use

- Designing a new platform, backend, service, worker, or serverless application from scratch.
- Defining bounded contexts and a context map for a domain.
- Deciding how to organize a module's folders and files.
- Choosing between monolith, modular monolith, and separately deployed services.
- Integrating with — or decoupling from — an external vendor (ERP, payment, storage, AI, identity, ticketing, workflow engine).
- Making external or inter-module calls resilient.
- Reviewing or auditing an existing codebase's module boundaries.
- Splitting a monolith: sizing components, finding duplication, measuring coupling, grouping into domains.

## When NOT to use

- Simple CRUD with a handful of endpoints — framework defaults are enough.
- Framework API syntax ("how do I add middleware in Fastify") — that is documentation, not architecture.
- Frontend/UI component architecture.
- Pure infrastructure capacity sizing or cost planning.
- Phased extraction roadmaps as the *primary* deliverable — this skill produces the analysis those roadmaps depend on.

---

## The one rule that governs everything

**Separate the logical boundary (always strong) from the physical boundary (evolutionary).**

Modules, contracts, state ownership, and the Anti-Corruption Layer are non-negotiable from day one. Whether a module gets its own deploy, its own database, or its own repository is an **operational** decision made later — never a structural prerequisite. This is what lets a system start as one deployable unit and grow without a rewrite.

A useful mental image: the house has well-divided rooms (modules). Inside each room, things sit out in the open grouped by what they are for (flat-by-aggregate) — not buried in nested drawers (technical-layer folders). The floor plan (boundaries) is what matters most.

**Default:** one deployable unit, fewer boundaries, strong contracts. Add granularity only when the product, the load, or the team justifies it — never because "it feels big" or because microservices are fashionable.

---

## Runtime-shape neutrality

The same modular structure maps onto every runtime. Only the **composition root** and the **inbound/outbound adapters** change; the domain, the module contracts, and the state ownership rules do not.

| Concept | HTTP server | Serverless function | Queue / stream worker | CLI or batch job | Scheduled job |
| --- | --- | --- | --- | --- | --- |
| **Composition root** | app bootstrap file | module-scope init *outside* the handler (reused across warm invocations) | consumer bootstrap | `main()` | scheduler entry point |
| **Inbound adapter** | controller / route handler | function handler | message handler | argument parser | trigger handler |
| **Module public surface** | facade exported from the module's index | identical | identical | identical | identical |
| **Application layer** | service / use case | service / use case | service / use case | service / use case | service / use case |
| **Outbound adapter** | HTTP client, repository | same (watch cold-start cost) | same | same | same |
| **Event transport** | in-process bus + outbox relay | managed bus (SNS/SQS/EventBridge/PubSub) | broker subscription | usually a direct call | usually a direct call |
| **Outbox relay** | background worker in the same process | scheduled function polling the outbox table | dedicated relay consumer | n/a | the job itself can be the relay |
| **Real-time push** | SSE (default) or WebSocket | managed WebSocket API or push service | n/a | n/a | n/a |
| **Config** | environment variables | environment variables / parameter store | environment variables | flags + env | env |

**Rules that hold in every shape:**

- The composition root is the *only* place that knows which concrete adapter implements which port. Keep it thin — wiring, no logic.
- A handler (HTTP, queue, cron, CLI) is a **transport adapter**. It parses input, calls one application operation, and maps the result. No business rules live there.
- A module must not assume it shares a process with another module. If it works in-process today, it should work across a queue tomorrow with an adapter swap.
- Serverless caveat: put expensive construction (connection pools, SDK clients, adapter graphs) at module scope so warm invocations reuse it — but never keep *mutable request state* there.
- Worker caveat: a worker with no HTTP surface still has a public contract — the message schemas it consumes and publishes. Version them like an API.

---

## Reference map

Load a reference only when the current phase needs it. Do not load them all at once.

| Topic | What it covers | Reference |
| --- | --- | --- |
| Principles | 10 modular (P1–P10) + 9 structural (P11–P19), each with agent rules and examples, plus the conflict hierarchy | `references/principles.md` |
| Domain & boundaries | DDD strategic: subdomain classification, ubiquitous language, bounded contexts, context map, integration patterns, cohesion scoring, sizing, worked examples | `references/domain-boundaries.md` |
| Module internals | Flat-by-aggregate layout, suffixes, depth, flat-vs-subdomain test, facades, sub-units, layering, tactical DDD building blocks, CQRS-when | `references/module-internals.md` |
| Contracts & communication | Ports & Adapters / ACL, sync vs async, events, transactional outbox, event naming, idempotent consumers, transport selection, real-time push | `references/contracts-and-communication.md` |
| State isolation | Ownership rules, naming, schema strategies, cross-context references, transaction ownership, detection commands | `references/state-isolation.md` |
| Resilience | Timeout → breaker → retry, backoff + full jitter, idempotency keys, retry budget, bulkhead, fallbacks, durable execution | `references/resilience.md` |
| Coupling analysis | Khononov's three-dimensional model (strength, distance, volatility), connascence, balance formula, diagnosis table, report format | `references/coupling-analysis.md` |
| Decomposition | The ordered Patterns 1–5 pipeline: size, common-domain duplication, flattening, coupling, domain grouping | `references/decomposition-pipeline.md` |
| Validation & testing | Fitness functions, compliance/audit pass, severity tiers, test levels per layer, mock strategy | `references/validation-and-testing.md` |
| Technology selection | Choosing per capability (transport, persistence, bus, cache, real-time, observability) with a worked modern stack and per-runtime notes | `references/technology-selection.md` |
| Architecture document | Producing the architecture write-up: standard sections, diagram catalog, diagram conventions | `references/architecture-document.md` |

---

## Workflow A — Designing (greenfield or a new module)

Move through phases in order; each has an exit criterion. State assumptions explicitly; if the domain is unclear, ask before guessing.

### Phase 0 — Frame the scope

Confirm: new platform, new module in an existing one, or a review? Confirm the **runtime shape** (HTTP service, worker, serverless, CLI, hybrid) and any hard constraints (existing stack, cloud, team size, compliance). Default to **one deployable unit** unless a hard constraint says otherwise.

*Exit:* scope, runtime shape, and constraints written down.

### Phase 1 — Domain discovery (DDD strategic)

Read `references/domain-boundaries.md`. Identify subdomains from the **business language**, classify each as **Core / Supporting / Generic**, and capture the ubiquitous language of each. Never group by technical layer. If multiple bounded-context interpretations exist, present them — do not pick silently.

*Exit:* a list of candidate bounded contexts, each classified, with a one-line responsibility and its key aggregates.

### Phase 2 — Boundaries, context map, and state ownership

Draw the context map: which contexts exist, how they relate (Customer/Supplier, Conformist, Open Host Service, Published Language, Shared Kernel, Anti-Corruption Layer), and which are Core. Then decide **state ownership**: one database is fine, but each module is the **sole writer of its own tables**. No foreign keys across module boundaries — reference other contexts by id. Keep transactionally cohesive aggregates together; do not over-split a Core.

Read `references/state-isolation.md` for the ownership and naming rules.

*Exit:* context map + a state-ownership map (one module = its tables/collections/streams).

### Phase 3 — Module internals

Read `references/module-internals.md`. Inside each module, organize by **aggregate**, not by technical layer: 1 business concept = 1 folder; technical layers become file **suffixes** (`.entity`, `.service`, `.repository`, `.controller`). Keep depth ≤ 2 (flat) or ≤ 3 (subdomain-based). The dependency rule still holds (presentation → application → domain ← infrastructure) — it is expressed by suffixes and co-location, not by layer folders. Apply the 6-criteria test to decide flat vs subdomain; default to flat.

*Exit:* a folder layout per module and a flat-vs-subdomain decision with rationale.

### Phase 4 — Contracts, ACL, and communication

Read `references/contracts-and-communication.md`. Every external system goes **behind a Port + Adapter (ACL)** — including internal services owned by another team. The domain declares ports in its own language; adapters translate the external model in and out.

Between internal modules: **synchronous only inside one aggregate** (one atomic transaction); **events via the transactional outbox** across modules, with idempotent consumers. Add real-time push (SSE by default) when the UX needs it.

*Exit:* the list of ports + adapters; the list of domain events named `module.aggregate.action`; sync-vs-async decisions with their consistency and failure semantics.

### Phase 5 — Resilience

Read `references/resilience.md`. Wrap every external and inter-service call: **timeout → circuit breaker → retry (capped exponential backoff + full jitter)**. Retry only idempotent operations; writes carry an idempotency key. Cap retries with a budget (~10% of traffic), trip the breaker on a sliding-window error rate, and give every breaker a **named fallback**. Add durable execution only when a long-running, multi-step process must survive restarts.

*Exit:* a resilience policy per adapter, plus idempotency keys for every write.

### Phase 6 — Technology and evolution

Read `references/technology-selection.md`. Choose concrete technology per capability and state the trade-offs. Then state the **evolution path**: stage 1 (one deploy, clear boundaries, events and outbox already in place) is the current state; promote a module to its own deployable/database only when *its own* metrics justify it.

*Exit:* a technology table + an evolution note.

### Phase 7 — Document the architecture

Read `references/architecture-document.md`. Produce the architecture write-up: overview, domains, principles, module map, bounded contexts, ACL, communication, module internals, data model with ownership, resilience, real-time, evolution, and the technology table — with one focused diagram per section where a diagram earns its place.

*Exit:* a coherent architecture document with contiguously numbered diagrams.

---

## Workflow B — Reviewing or decomposing an existing system

Read `references/decomposition-pipeline.md` and run **Patterns 1 → 5 in order**. Each step feeds the next; do not skip one unless the user narrows scope explicitly, and if they do, say which patterns were skipped and how that limits the conclusions.

| Step | Pattern | Produces |
| --- | --- | --- |
| 1 | Identify and size components | Component inventory with statements, files, %, and outliers |
| 2 | Common domain detection | Duplicated domain functionality + a consolidation plan with coupling impact |
| 3 | Flattening / hierarchy | Orphaned classes in root namespaces + a flattening strategy per namespace |
| 4 | Coupling analysis | Annotated dependency graph with strength/distance/volatility and a balance score (`references/coupling-analysis.md`) |
| 5 | Domain identification and grouping | Components grouped into domain-aligned candidate units + a namespace refactoring plan |

Ground **Pattern 5** in business language by reading `references/domain-boundaries.md` before or alongside it — structure alone produces solution-space groupings, not linguistic boundaries.

Always carry outputs forward and cite concrete evidence: file paths, module names, import edges, tables. Generic advice is a failed review.

For a **compliance/audit pass** (rather than a decomposition), use the signals and severity tiers in `references/validation-and-testing.md`.

**Pattern 6 (extraction and phased roadmap) is deliberately out of scope.** After Pattern 5, planning the extraction order, milestones, and migration strategy is a separate deliverable.

---

## Decision cheat sheet

- **Subdomain class:** competitive advantage → Core; business-specific but not differentiating → Supporting; solved problem you consume → Generic.
- **Tactical intensity:** full (rich aggregates, value objects, invariants, events) in Core; moderate in Supporting; minimal in Generic. Forcing richness where there is no invariant is over-engineering, not rigor.
- **Flat vs subdomain:** flat by default (depth 2). Go subdomain-based (depth 3) only when 4+ of 6 hold: different personas, different authorization, different execution model, different scaling, independent deployability, independent failure.
- **Split vs merge:** default to **fewer boundaries** until real pain appears. Favor splitting when several of the six criteria hold (language, rate of change, scale/SLO, consistency, ownership, observable pain).
- **Sync vs event:** synchronous inside one aggregate (atomicity matters); event via outbox across modules (tolerates eventual consistency).
- **Consolidate duplicated logic?** Only after computing afferent coupling before and after. Same total coupling → safe. Large increase → prefer a shared library, or accept the duplication.
- **CQRS?** Not by default. Only when read and write patterns genuinely diverge (read-heavy optimized queries, complex writes, different scaling).
- **Durable execution?** Only for long-running, multi-step processes that must resume exactly where they left off after a crash.
- **Coupling balance:** `BALANCE = (STRENGTH XOR DISTANCE) OR NOT VOLATILITY`. Things that change together should live together; distant things must be loosely coupled.

---

## Hard rules (non-negotiable)

- Each module **writes only its own state**. Cross-context links are by id, validated in code — never a foreign key across a boundary.
- **No direct cross-module imports of internals.** Communicate through a facade/contract or an event. Never import another module's entities, repositories, or internal services.
- **Every external system sits behind a port + adapter (ACL).** No vendor type leaks into the domain.
- **Retry only idempotent operations.** Writes carry an idempotency key and the server dedupes.
- **Every cross-module and external call has an explicit timeout and failure behavior.** "Hang forever" is a design bug at a boundary.
- **Business rules never live in transport adapters** (HTTP handlers, queue handlers, CLI parsers, UI).
- **Names are unambiguous across modules.** Prefix persisted concepts with their module (`BillingPlan`, not `Plan`).
- **Modular principles (P1–P10) beat structural ones (P11–P19).** If co-locating would force a cross-module entity import, use a facade + DTO. If splitting an aggregate would break a transaction, keep it together.

---

## Behavioral guidelines

These govern *how* to work, not just what to build.

**Think before coding.** State assumptions about domain boundaries explicitly. If multiple bounded-context interpretations exist, present them. If a simpler structure would do, say so and push back when warranted. If the domain is unclear, stop and ask.

**Simplicity first.** Design the minimum viable architecture. No CQRS without distinct read/write patterns. No event sourcing without a real audit requirement. No abstraction for single-use code. If 3 modules suffice, do not create 8.

**Surgical changes.** In an existing codebase, do not "improve" adjacent modules outside the task. Match existing style and conventions even where you would do it differently. Mention unrelated issues; do not silently fix them.

**Goal-driven execution.** Every architectural decision gets a verifiable success criterion. "Add a module" → "module owns its state, exposes one facade, has no cross-module imports, tests pass." "Fix communication" → "events flow, no direct cross-module service calls remain."

**Evidence over assertion.** In reviews, every finding cites a path, an import, a table, or a metric.

---

## Anti-patterns / red flags

- ❌ Separate services or separate databases from day one because "it feels big."
- ❌ Technical-layer folders inside a module (`core/service/`, `http/controller/`, `persistence/entity/`).
- ❌ A module reading or writing another module's tables ("reach-through persistence").
- ❌ A central persistence layer that registers and exposes *every* store for *every* module.
- ❌ Vendor SDK types leaking into the domain (no ACL).
- ❌ Business rules in transport adapters; adapters talking straight to storage.
- ❌ Writes that span module boundaries with no saga, outbox, or single-owner rule.
- ❌ Repositories, internal services, or persistence types exported as the module's public API.
- ❌ "Facades" that query, map, or hold policy instead of delegating.
- ❌ Retrying non-idempotent writes; retries without backoff + jitter; retries with no budget; retries stacked at multiple layers.
- ❌ Splitting a cohesive transactional Core into event-coupled fragments.
- ❌ Forcing rich tactical DDD onto a Generic subdomain that has no invariants.
- ❌ Bounded contexts drawn around technical layers, single entities, one-per-database, or the org chart.
- ❌ Generic entity names (`User`, `Plan`, `Item`) with no module prefix.
- ❌ Shared mutable global state across boundaries.
- ❌ In-memory-only event delivery used for production inter-module communication.

---

## Quick checklist (before proposing or approving a structure)

- [ ] Every module's public surface is minimal and listable without naming storage or internal services.
- [ ] Names and state ownership are unambiguous per module; one writer per fact.
- [ ] No cross-module persistence shortcuts without an explicit, documented contract.
- [ ] Business rules sit in domain/application code, not in adapters.
- [ ] Every cross-module call has explicit timeout, retry, and failure semantics.
- [ ] Async consumers are idempotent or deduplicated.
- [ ] Observability can answer "which module failed and why?" without spelunking.
- [ ] Every external dependency sits behind a port; the composition root is the only place that names concrete adapters.
- [ ] If a context has sub-units: each owns its slice; there is no "registers everything" persistence grab-bag.
- [ ] The evolution path is stated — what would have to be true to split further.

---

## Examples

**Example 1 — Design a new platform.** *"Design the architecture for an accounts-payable platform that posts entries into the customer's ERP."* → Phase 1: discover and classify subdomains (Payables and Operations as Core; Documents/Audit as Supporting; Identity/Notifications as Generic). Phase 2: context map + one database, table-per-module ownership. Phase 3: flat-by-aggregate layout per module. Phase 4: ERP/ticketing/storage behind ports + adapters; cross-module events via outbox; SSE for the operator queue. Phase 5: resilience policy on the ERP adapter with `idempotency_key = payment.id`. Phase 6: technology table + a "one deploy now" evolution note.

**Example 2 — Organize a module.** *"How do I organize the billing module?"* → Read `module-internals.md`; propose `billing/subscription/`, `billing/invoice/`, `billing/payment/` with co-located files and a `__test__/` folder each; run the 6-criteria test (billing scores 1/6 → stays flat). Show the dependency rule preserved via suffixes.

**Example 3 — Decouple from a vendor.** *"I don't want to be locked into this ERP."* → Read `contracts-and-communication.md`; define `ErpLedgerPort` in domain language ("post a ledger entry", "fetch the next demand"); implement `OmieAdapter` translating at the boundary; show that swapping vendors means writing one new adapter. The domain never imports vendor types.

**Example 4 — Split a monolith.** *"We want to break this monolith up."* → Workflow B: Pattern 1 inventory (Reporting is 33% of the codebase, >2σ → split candidate); Pattern 2 finds three near-duplicate notification components with unchanged total afferent coupling → safe to consolidate; Pattern 3 finds 45 orphaned files in the ticket root namespace → split up; Pattern 4 flags symmetric functional coupling between two services owned by different teams → CRITICAL; Pattern 5 groups components into 5 domains with a namespace refactoring plan.

**Example 5 — Serverless.** *"We're on Lambda — does any of this apply?"* → Yes, unchanged. Bounded contexts become function groups sharing a module library; each handler is a transport adapter calling one application operation; the composition root moves to module-scope init; the outbox relay becomes a scheduled function polling the outbox table; events go over the managed bus. State ownership, ACL, idempotency, and contracts are identical.

---

## Sources & attribution

Compiled from four skills, preserving their architectural content:

- **evolutionary-modular-architecture** and **nestjs-modular-monolith** — Felipe Rodrigues (github.com/felipfr), CC-BY-4.0. Source of the 19 principles, flat-by-aggregate, ACL + outbox, resilience, and the architecture-document guidance.
- **modular-design-principles** — technology-agnostic principle depth, the bounded-context workflow, split/merge criteria, sub-units, and the compliance pass.
- **modular-decomposition** — the Patterns 1–5 pipeline and the embedded DDD strategic analysis.

Underlying literature: Evans, *Domain-Driven Design* (2003); Vernon, *Implementing DDD* (2013); Khononov, *Balancing Coupling in Software Design*; Grzybek / Drotbohm on modular monoliths; R. Martin (Screaming Architecture); Bogard (Vertical Slice); the AWS architecture blog on backoff and jitter.
