# Module Internals Reference

How to organize a module *inside* its boundary. The goal: minimize discovery cost while preserving the dependency rule. Language- and framework-neutral — the suffixes shown use `.ts` for concreteness, but the pattern is identical for `.py`, `.go`, `.java`, `.rb`, `.cs`.

## Table of Contents

1. The principle: flat-by-aggregate
2. Why flat (discovery cost)
3. Flat package structure (depth 2)
4. Subdomain-based structure (depth 3)
5. Flat vs subdomain: the 6-criteria test
6. Facades and the public API
7. The dependency rule, expressed as suffixes
8. Tactical DDD building blocks
9. Tactical golden rules
10. Tactical intensity by subdomain
11. Domain errors
12. Services vs CQRS
13. Tests inside a module
14. Scaffolding a new module or aggregate

---

## 1. The principle: flat-by-aggregate

**1 business concept = 1 folder.** All production files of an aggregate live together; technical layers become file **suffixes**, not folders. `ls module/` reveals the domain (Screaming Architecture), not the framework.

```
✅ billing/subscription/subscription.{entity,repository,service,controller,types}.ts
❌ billing/core/service/  +  billing/persistence/entity/   ← legacy technical layers
```

This is a modern application of established patterns — Modular Monolith, Bounded Context, Screaming Architecture (R. Martin), Vertical Slice (Bogard), Package by Feature — not a replacement for them.

---

## 2. Why flat (discovery cost)

Code is now read by humans **and** by AI agents. Depth and indirection turn directly into more tokens, more tool calls, and a higher error rate. Keeping a concept in one folder cuts that cost sharply — practical estimates report 30–50% fewer tokens on read/refactor tasks.

It also speeds human onboarding (one mental model: "1 concept = 1 folder") and makes the structural checks in PR review automatable.

---

## 3. Flat package structure (depth 2)

Use for a single cohesive domain (3–8 aggregates). Examples: billing, identity, notifications.

```
<module>/
├── <aggregate>/
│   ├── <aggregate>.entity.ts          # domain model + behavior
│   ├── <aggregate>.repository.ts      # infrastructure: implements the domain port
│   ├── <aggregate>.service.ts         # application: the default unit of behavior
│   ├── <aggregate>.controller.ts      # presentation (or .handler.ts / .resolver.ts / .command.ts)
│   ├── <aggregate>.types.ts
│   ├── <aggregate>.dto.ts
│   └── __test__/
│       └── <aggregate>.service.spec.ts
├── shared/persistence/                # connection / datasource only — zero domain repos
├── <module>.module.ts                 # or wiring.ts / container.ts / index of providers
├── <module>.facade.ts                 # the public surface
├── config.ts
└── index.ts                           # exports facade + module wiring ONLY
```

The transport file is named for the runtime shape: `.controller.ts` (HTTP), `.resolver.ts` (GraphQL), `.handler.ts` (queue/serverless), `.command.ts` (CLI), `.job.ts` (scheduled). The rest of the aggregate is unchanged.

---

## 4. Subdomain-based structure (depth 3)

Use when a module has multiple subdomains with independent scaling or failure needs (10+ aggregates). Examples: content (management/catalog), analytics (ingestion/aggregation/reporting).

```
<module>/
├── <subdomain>/
│   ├── <aggregate>/
│   │   ├── <aggregate>.entity.ts
│   │   ├── <aggregate>.repository.ts
│   │   ├── <aggregate>.service.ts
│   │   └── __test__/
│   ├── <subdomain>.module.ts          # registers its OWN repos + services
│   └── <subdomain>.facade.ts          # pure delegation, exported to siblings
├── shared/
│   ├── contract/                      # queue / event payload types
│   ├── enum/
│   └── persistence/                   # connection only — zero repos
├── <module>.module.ts                 # composes subdomains
├── <module>.facade.ts                 # composes subdomain facades
└── index.ts
```

**Rules:** each subdomain owns its repositories; `shared/persistence` holds only the connection; cross-subdomain reads go through the sibling's **internal facade**, never its repositories.

---

## 5. Flat vs subdomain: the 6-criteria test

Default to **flat**. Go subdomain-based only when **4+ of 6** hold:

1. Different user personas (admin vs customer)?
2. Different authorization models?
3. Different execution model (request/response vs queue vs stream vs batch)?
4. Different scaling characteristics (read-heavy vs write-heavy, CPU vs I/O)?
5. Could it be deployed independently?
6. Can it fail in isolation?

**Decision matrix:**

| Cohesion | Coupling | Verdict |
| --- | --- | --- |
| High | Low | Strong subdomain candidate |
| High | High | **Keep flat** — the coupling means they belong together |
| Low | any | **Refactor first**, do not split |

**Red flags (do NOT split):** "it feels big" · "to make code easier to find" (use aggregate naming, not layer folders) · tightly coupled features · matching the org chart.

**Aggregate-level thresholds:** a single aggregate over ~15 files is a review signal and over ~25 files is a strong split candidate (split into sub-aggregates within the same package). A flat package over ~8 aggregates with low coupling → consider subdomains.

---

## 6. Facades and the public API

The pattern is always **Facade → Service → Repository**.

- The facade **only delegates**. No querying, no mapping, no business logic, no policy.
- The package `index` exports **only** the facade and the module wiring — never services, repositories, controllers, entities, or persistence types.
- Export nothing unless another module genuinely needs it. When in doubt, export nothing and add the operation later.
- Cross-module communication goes through the facade contract or an event — not through exports of internals.

```ts
// billing.facade.ts — the entire public surface of the billing module
export interface BillingFacade {
  activateSubscription(cmd: ActivateSubscription): Promise<SubscriptionSummary>
  getSubscriptionSummary(id: string): Promise<SubscriptionSummary | null>
}
```

`SubscriptionSummary` is an **integration DTO**, not the internal entity. See `coupling-analysis.md` §Contract Coupling.

---

## 7. The dependency rule, expressed as suffixes

Clean Architecture's dependency rule is preserved without layer folders. Within one aggregate folder:

```
<x>.controller.ts   (presentation)  ──depends on──▶
<x>.service.ts      (application)   ──depends on──▶
<x>.entity.ts  +  the repository INTERFACE   (domain)
                                             ▲
<x>.repository.ts   (infrastructure) ──implements──┘
```

Dependencies still point **inward** toward the domain; the layers are just suffixes in one folder instead of separate directories.

**Layer responsibilities:**

- **Domain** (innermost, no external dependencies): entities, value objects, aggregates, domain events, domain errors, and repository/port *interfaces*.
- **Application**: orchestrates the domain; defines use cases; owns transactions; publishes events. Knows domain, not infrastructure.
- **Infrastructure**: implements the domain's interfaces — repositories, external-service adapters, message publishers. Depends inward.
- **Presentation / transport**: controllers, resolvers, queue handlers, CLI commands. Parse input, call one application operation, map the result. **No business rules.**

If your framework or team requires literal layer folders, keep the same ordering — the rule is about direction of dependency, not directory names.

### Illustrative shapes

```ts
// domain — the port lives with the domain, in domain language
export interface BillingPlanRepository {
  findById(id: string): Promise<BillingPlan | null>
  save(plan: BillingPlan): Promise<BillingPlan>
  delete(id: string): Promise<void>
}

// application — orchestration, transaction, event
class BillingPlanService {
  constructor(private repo: BillingPlanRepository, private events: EventPublisher) {}

  async create(name: string, priceInCents: number, interval: BillingInterval) {
    const plan = BillingPlan.create(name, priceInCents, interval)   // invariants enforced inside
    const saved = await this.repo.save(plan)
    await this.events.publish('billing.plan.created', { planId: saved.id, name: saved.name })
    return saved
  }
}

// infrastructure — implements the domain port, maps persistence rows to domain objects
class SqlBillingPlanRepository implements BillingPlanRepository { /* toDomain(row) ... */ }

// presentation — thin; parses, delegates, maps
async function createPlanHandler(input) {
  const plan = await billingPlanService.create(input.name, input.priceInCents, input.interval)
  return BillingPlanResponse.from(plan)
}
```

The same four shapes exist in a Lambda (handler replaces controller), a queue worker (message handler replaces controller), and a CLI (argument parser replaces controller).

---

## 8. Tactical DDD building blocks

- **Entity** — identity tracked over time; **rich behavior**. Methods speak the ubiquitous language: `approve()`, `cancel()`, `updateProfile()` — not `setStatus()`.
- **Value Object** — defined by its attributes, immutable, no identity (`Money`, `Cnpj`, `Email`, `DueDate`). Validates itself in its constructor. **Prefer value objects over entities.**
- **Aggregate** — a root entity plus children sharing invariants; external access only through the root. Keep aggregates small.
- **Domain Event** — an immutable fact named `module.aggregate.action`; serializable payload; references other aggregates by id only.
- **Domain Service** — an operation that spans aggregates or belongs to none. Use sparingly; too many domain services means an anemic model.
- **Repository** — a collection-like interface *declared by the domain*, implemented by infrastructure.

```ts
// Value object: immutable, self-validating, no identity
class Money {
  constructor(readonly amount: number, readonly currency: string) {
    if (amount < 0) throw new InvalidMoneyError('Amount cannot be negative')
    if (currency?.length !== 3) throw new InvalidMoneyError('Invalid currency code')
  }
  add(other: Money): Money {
    if (this.currency !== other.currency) throw new CurrencyMismatchError()
    return new Money(this.amount + other.amount, this.currency)
  }
  equals(other: Money) { return this.amount === other.amount && this.currency === other.currency }
}

// Aggregate: the root protects the invariants; children are reached only through it
class OrderAggregate {
  private items: OrderItem[] = []
  constructor(readonly id: string, readonly customerId: string, private status = OrderStatus.PENDING) {}

  addItem(productId: string, quantity: number, price: Money): void {
    if (this.status !== OrderStatus.PENDING) throw new OrderNotModifiableError(this.id)
    const existing = this.items.find(i => i.productId === productId)
    if (existing) existing.updateQuantity(existing.quantity + quantity)
    else this.items.push(new OrderItem(generateId(), productId, quantity, price))
  }

  confirm(): void {
    if (this.items.length === 0) throw new EmptyOrderError(this.id)   // last line of defense
    this.status = OrderStatus.CONFIRMED
  }

  get total(): Money { return this.items.reduce((s, i) => s.add(i.subtotal), new Money(0, 'USD')) }
}
```

Note `customerId: string` — a reference by id to another aggregate, never an object reference.

---

## 9. Tactical golden rules

1. **Behavior with data** — objects own their state *and* the operations that change it.
2. **Ubiquitous language in method names** — not CRUD verbs.
3. **Small aggregates** — root + value objects by default; add child entities only for a true shared invariant.
4. **One transaction = one aggregate** — cross-aggregate rules use eventual consistency via domain events.
5. **Reference by id** — never hold object references to other aggregates.
6. **Value objects first** — entities only when identity is essential.
7. **Protect invariants** — the aggregate is the last line of defense; never trust the caller.

---

## 10. Tactical intensity by subdomain

Apply tactical depth **in proportion**. Forcing rich models where there is no invariant is abstraction for its own sake.

| Subdomain type | Tactical intensity | What it looks like |
| --- | --- | --- |
| **Core** | Full | Rich aggregate, value objects, protected invariants, domain events |
| **Supporting** | Moderate | A few value objects and a small aggregate where a real rule exists |
| **Generic** | Minimal | Thin model / pure state holder; often just wraps an external service via an ACL |

"Not anemic" applies most strongly to the **Core**. A Generic context that only forwards to an external provider has no behavior to encapsulate — a lean model there is correct, not a smell. Anemia is a problem only when behavior that belongs in the model is scattered into services. Ref: Evans (2003); Vernon (2013).

---

## 11. Domain errors

Domain code raises **domain-specific errors**, never transport errors. A single boundary layer maps them to the transport's vocabulary (HTTP status, exit code, message nack/DLQ).

```ts
class DomainError extends Error {
  constructor(message: string) { super(message); this.name = this.constructor.name }
}
class OrderNotFoundError extends DomainError { constructor(id: string) { super(`Order ${id} not found`) } }
class OrderNotModifiableError extends DomainError { constructor(id: string) { super(`Order ${id} cannot be modified`) } }
```

**Mapping at the boundary.** Keep one registry that maps domain error → transport result, and let each module register its own errors at startup. This keeps the domain free of transport concerns and keeps the mapping in one auditable place.

```
registry.register(OrderNotFoundError,     → 404 / exit 4 / nack-no-retry)
registry.register(OrderNotModifiableError,→ 409 / exit 5 / nack-no-retry)
unmapped error                            → 500 / exit 1 / retry-then-DLQ  (and log with stack)
```

Per runtime: HTTP → status code + problem body with timestamp and path; queue worker → ack (permanent failure, send to DLQ) vs nack (transient, let the retry policy run); CLI → exit code + stderr message; scheduled job → structured failure record.

Validate **all** inputs at the transport boundary before they reach the application layer (schema validation on the DTO), so the domain only ever sees well-formed data.

---

## 12. Services vs CQRS

**Default: the simple service pattern.** A service groups an aggregate's actions, takes its dependencies through injection, and is called directly by the transport adapter. Sub-types of a service (state machine, validator, calculator) use **suffixes** — `.state-machine.service.ts` — not folders.

**CQRS is NOT the default.** Use it only when the domain genuinely benefits from separating read and write models.

| Use CQRS when | Do NOT use CQRS when |
| --- | --- |
| Read and write patterns differ significantly | Simple CRUD operations |
| Read-heavy workloads need optimized queries | Read and write models are identical |
| Complex domain logic on writes, simple retrieval on reads | Small dataset, uniform access patterns |
| Reads and writes need different scaling | The team is unfamiliar with the pattern |

With CQRS the application layer becomes **commands + queries + handlers** instead of services; the domain, infrastructure, and presentation layers are unchanged (the transport adapter dispatches to a bus rather than calling a service). The query side may bypass the domain model entirely and read a denormalized projection — that is the point of it.

**When to upgrade:** if a service grows large with clearly divergent read and write logic, or reads need independent optimization (denormalized views, caching layers), extract command and query handlers then — not before.

Similarly: **no Event Sourcing** unless a durable audit trail is a real requirement, and **no abstraction for single-use code**.

---

## 13. Tests inside a module

- **Unit tests** live in `<aggregate>/__test__/<file>.spec.ts` — next to the aggregate, not beside the production files.
- **End-to-end tests** are centralized per flow: `__test__/e2e/<flow>.e2e-spec.ts`.
- Domain entities are tested **purely** — no framework, no test container, no mocks.
- Services are tested against **mocked repository interfaces**, never against the real persistence library.

Full test-level guidance, mock strategy, and what to assert per layer: `validation-and-testing.md`.

---

## 14. Scaffolding a new module or aggregate

When adding a **module** (bounded context):

1. Name it from the ubiquitous language; write its one-line responsibility and what is explicitly out of scope.
2. Create the folder with `index.ts`, `<module>.facade.ts`, module wiring, and `config.ts`.
3. List the state it owns (tables/collections/streams), prefixed with the module name.
4. Declare its ports for anything external.
5. Declare the events it publishes and the events it consumes.
6. Add the first aggregate.

When adding an **aggregate**:

1. One folder named for the business concept (singular).
2. Entity with behavior in the ubiquitous language; value objects for the attributes with rules.
3. Repository interface in the domain; implementation with the `.repository` suffix.
4. Service with the aggregate's actions.
5. Transport file matching the runtime shape.
6. `__test__/` with at least the entity's invariant tests and the service's happy/error paths.
7. If it needs anything from another module: facade call or event — never a direct import.

**Order of delivery when implementing a full module:** domain entities → repository interface → service (or commands/queries) → DTOs with validation → repository implementation → transport adapter → module wiring with explicit exports → tests → domain events (if cross-module communication is needed).
