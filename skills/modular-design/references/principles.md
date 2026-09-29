# Principles Reference

19 principles: **10 modular** (P1–P10 — the foundation, about boundaries *between* units) + **9 structural** (P11–P19 — internal organization *inside* a unit). Modular principles always win conflicts with structural ones.

Everything here is technology-agnostic. Map it to your stack locally: "module" may be a package, a library, a folder, a namespace, a Lambda group, or a repository; "state" may be tables, collections, buckets, streams, or files.

## Table of Contents

1. Layered mental model
2. Modular principles (P1–P10), in depth
3. Structural principles (P11–P19)
4. Conflict hierarchy
5. Typical violations

---

## 1. Layered mental model

- **Composition roots** (applications, hosts, runners, function entry points): wire modules together. Keep orchestration thin — wiring, no logic.
- **Modules / bounded contexts**: cohesive units of behavior and data ownership; each should be understandable and testable on its own.
- **Shared kernels** (use sparingly): only stable, truly cross-cutting concepts. Resist turning them into a grab-bag of "everything everyone needs."

How you physically lay this out (monorepo, multi-repo, packages, libraries, functions) is a **delivery choice**, not the definition of modularity. The principles apply either way.

---

## 2. Modular principles (P1–P10), in depth

Each principle has a **definition**, **rules for agents**, and an **abstract example**. Criticality noted where it matters most.

### P1 — Well-Defined Boundaries (High)

**Definition.** Each module has clear responsibilities and exposes only a **small, intentional public surface** (operations, events, types that are part of the contract) from a single entry point. Everything else is implementation detail. Never import another module's internal classes; never share persisted entities.

**Rules for agents.**

- Prefer extending behavior by adding to the **documented** API rather than importing internals.
- When suggesting refactors, preserve or shrink the public surface; do not widen it "for convenience."
- Name things so **contract vs internal** is obvious in review, even without tooling to enforce it.

**Abstract example.** A "Checkout" context exposes `placeOrder(command)` and an `OrderPlaced` event. Other contexts must not reach into Checkout's internal pricing tables; they subscribe to the event or call `placeOrder` — never "update row X."

---

### P2 — Composability (Medium)

**Definition.** Modules are building blocks: the same modules can be **assembled into different products or deployments** without rewriting their core logic for each combination (one deploy today, several later).

**Rules for agents.**

- Avoid hidden assumptions like "this only runs when module B is present" unless expressed as an **optional integration** or plugin contract.
- Configuration and feature flags must not become spaghetti that only one deployment understands.

**Abstract example.** The same "Inventory" module works in a small CLI tool and a large web app because its contract does not assume a specific UI or host — only the composition root changes.

---

### P3 — Independence (High)

**Definition.** Modules build, test, and deploy in isolation and do not rely on **hidden shared mutable state** across boundaries. Tests can run a module with **fakes** at its edges. Modules communicate via interfaces/events, never by direct method calls into another module.

**Rules for agents.**

- Flag "global singletons" that encode cross-module policy without an explicit contract.
- Prefer **passing dependencies explicitly** or declared injection over ambient globals for cross-cutting concerns.

**Abstract example.** Two services in different modules both mutate a process-wide cache keyed by "user id" without coordination → independence is violated. Replace with an explicit cache interface owned by one module, or a documented shared service.

---

### P4 — Individual Scale (Medium)

**Definition.** **Throughput, storage, batching, concurrency, and limits** can be tuned per module where it matters, without forcing one global setting on everyone or rewriting other modules.

**Rules for agents.**

- When performance tuning, ask **which bounded context** owns the bottleneck; avoid "fixing" it by coupling unrelated code paths.
- Suggest **per-module** quotas, pools, or batch sizes when load profiles differ.

**Abstract example.** "Search" needs a large read replica and aggressive caching; "Billing" needs strict serial writes. Their scaling policies are not identical, and neither module forces the other's settings.

---

### P5 — Explicit Communication (High)

**Definition.** All **cross-module** interaction goes through **known contracts**: APIs, messages, events, or versioned schemas — never incidental shared files, implicit side channels, or assumptions about another module's internals.

**Rules for agents.**

- Document **inputs, outputs, errors, and versioning** for anything that crosses a boundary.
- Discourage "just import this DTO from their package" when that DTO is really an **internal** persistence shape.

**Abstract example.** Module A notifies Module B via `OrderPlaced { orderId, placedAt }` on a bus — not by writing into B's database "because it's faster."

---

### P6 — Replaceability (Medium)

**Definition.** Dependencies on other modules and on external systems are expressed through **interfaces, protocols, or stable contracts**, so implementations can be swapped or mocked without touching consumers. Never export concrete classes as the module API.

**Rules for agents.**

- At boundaries, prefer **narrow interfaces** ("payment gateway", "clock", "id generator") over concrete vendor types leaking inward.
- Question refactors that **pin** a module to one technology everywhere, unless that is a deliberate platform choice.

**Abstract example.** "Notifications" depends on `Notifier` with `send(recipient, body)`; email vs SMS vs push is replaceable behind that port.

---

### P7 — Deployment Independence (Medium)

**Definition.** Module code does not **assume** co-location in the same process, host, or release cadence unless that is an **explicit** architectural decision. Deployment logic lives in apps/composition roots; environment variables carry deploy-specific config.

**Rules for agents.**

- Avoid "call this function directly in their package" as the only integration story when multiple deployment topologies are possible.
- Prefer contracts that work across **in-process, out-of-process, or async** delivery with minimal change.

**Abstract example.** The same domain logic runs in a monolith today and behind a message queue tomorrow, because interactions were modeled as operations and events, not as hardcoded in-process singletons.

---

### P8 — State Isolation (CRITICAL)

**Definition.** Each module **owns** its authoritative store and the naming of its facts. One shared database is acceptable, but **never** share tables, **never** put foreign keys across module boundaries, **never** read another module's repository. Reference other contexts by id. No silent sharing of the same logical data without a clear rule (who writes, who reads, how consistency is achieved).

**Rules for agents.**

- Treat **reach-through persistence** (reading/writing another module's store directly) as a design smell unless documented as an exceptional, reviewed pattern.
- Require **unambiguous names** for persisted concepts when multiple modules have similar nouns — prefix with the module (`BillingPlan`, not `Plan`).

**Abstract example.** "Customer" in CRM and "Customer" in Billing are different aggregates with different ids or an explicit mapping — not two modules updating one ambiguous `customers` row.

> P8 is usually the hardest. Ambiguous ownership of data or names is the most frequent source of "works until it doesn't" integration bugs. Depth: `state-isolation.md`.

---

### P9 — Observability (High)

**Definition.** Logs, metrics, traces, and health checks can be **attributed** to a module (and often to a use case), with correlation ids, so incidents are diagnosable without reading the whole system. Do not mix module concerns in telemetry.

**Rules for agents.**

- When adding diagnostics, include **context** (which module, which operation, which correlation id) — not only "error happened."
- Avoid log lines that **cannot** be filtered by owning team or subsystem.

**Abstract example.** A failed payment shows a `billing.capture` span with `orderId` and a clear error code; support does not grep unrelated modules' noise to find root cause.

---

### P10 — Fail Independence (High)

**Definition.** Failures are **bounded**: timeouts, retries with backoff, bulkheads, circuit breaking, idempotency, graceful degradation — so one module's outage does not cascade blindly.

**Rules for agents.**

- Cross-module calls need **explicit** timeout and failure semantics; "hang forever" is a design bug at the boundary.
- Async handlers must be **idempotent** or deduplicated wherever duplicates are possible.

**Abstract example.** When Recommendations is down, Checkout still completes using defaults or a cached tier; the UI degrades instead of blocking the purchase.

> Depth: `resilience.md`.

---

## 3. Structural principles (P11–P19)

These define how code is organized **inside** each module — optimized for humans *and* for AI agents (the discovery cost of a concept turns directly into tokens and tool calls).

- **P11 — Co-location by Aggregate (High).** One business concept = one folder. All production files of that concept (entity, repository, service, transport handler, DTO, types) live together. Unit tests live in `<aggregate>/__test__/`. Never split by technical layer.
- **P12 — Suffixes > Folders (Medium).** Use `.types`, `.dto`, `.constants` suffixes — not `types/` subfolders. `user.types.ts`, not `types/user.ts`.
- **P13 — Depth ≤ 2–3 (High).** Flat packages: depth 2 (`module/aggregate/file`). Subdomain-based: depth 3 (`module/subdomain/aggregate/file`). `shared/` is exempt.
- **P14 — Folder Only if ≥ 2–3 Cohesive Files (Medium).** No single-file folder; use a suffix instead. Exception: `__test__/` is a sanctioned semantic folder even with one file.
- **P15 — Aggregate Limits (Medium).** ~15 files in one aggregate = review signal; ~25+ = strong split candidate (split into sub-aggregates, or promote to a subdomain).
- **P16 — Flat Optimization for Discovery (Medium).** A flat structure minimizes discovery cost — `ls module/` reveals the **domain**, not the framework (Screaming Architecture).
- **P17 — No README Inside an Aggregate (Low).** README only at the package root; never inside aggregate/subdomain business folders.
- **P18 — Service as the Default Unit (Medium).** Services group an aggregate's actions; sub-types (state machine, validator, calculator) use suffixes (`.state-machine.service`), not folders. A separate "use-case" construct is unnecessary when co-location already provides the focus — add it only if your framework enforces strict layer separation.
- **P19 — Intentional Shared Kernel is Legitimate (Medium).** For behavior-less persistence entities, a **documented** shared kernel is an accepted DDD pattern when: sharing is justified by cross-subdomain reads, ownership is centralized and documented in the code, and the entity is a pure state holder. The anti-pattern is the **accidental** shared kernel — entities with no clear owner. Ref: Vernon, *Implementing DDD* (2013), ch. 3; Evans, *DDD* (2003), ch. 14.

---

## 4. Conflict hierarchy

**Modular (P1–P10) prevails over structural (P11–P19).** When structural convenience would break a modular boundary, modular wins.

| Conflict | Resolution |
| --- | --- |
| Co-locating files (P11) would require a cross-module entity import | Use a facade + DTO (P1, P5) — do not import entities |
| Splitting an aggregate (P15) would break a transactional boundary | Keep it together (P8); split by subdomain instead |
| Flat depth (P13) vs subdomain isolation (P3) | A subdomain layout at depth 3 is allowed when P3/P4 justify it |
| Suffix preference (P12) vs shared test helpers | `__test__/` is exempt |
| Individual scale (P4) vs one shared configuration | Per-module settings win; the composition root supplies them |

**Rule of thumb:** if obeying P11–P19 would break P1, P5, or P8, stop and use the modular pattern.

**Make it executable.** Turn these into automated checks rather than relying on review — see `validation-and-testing.md` for fitness functions covering structure (P11–P17) and boundaries/state isolation (P1, P3, P8).

---

## 5. Typical violations (stated abstractly)

1. **Colliding concepts** — the same name or schema for different things in different modules, or duplicate "global" definitions that diverge over time.
2. **Reach-through persistence** — one module reading or writing another module's tables, buckets, or documents **without** an agreed contract.
3. **Centralized data ownership** — a single persistence layer that registers and exposes **all** stores for **all** modules, encouraging hidden coupling.
4. **Logic at the edge** — business rules in transport adapters (HTTP handlers, queue handlers, UI, CLI) instead of domain/application code.
5. **Edge talking to storage directly** — adapters depending on low-level persistence APIs instead of use cases or application services.
6. **Unscoped transactions** — writes that span boundaries without clear transaction ownership and failure semantics.
7. **Leaky exports** — repositories, internal services, or implementation types exposed as the module's public API.
8. **Facades that aren't thin** — "public" entry points that embed querying, mapping, or policy instead of delegating to the right place inside the module.

---

Sources: Modular Monolith (Grzybek; Drotbohm / Spring Modulith); the 10 modular principles (Arquiteturas Modulares whitepaper, TechLeads.club; Ghemawat et al., HotOS 2023); structural principles validated in production (flat-by-aggregate refactor).
