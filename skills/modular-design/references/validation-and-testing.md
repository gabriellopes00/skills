# Validation & Testing Reference

Make the principles **executable** instead of relying on review, and test each layer at the level where its logic actually lives.

## Table of Contents

1. Fitness functions — structure
2. Fitness functions — boundaries and state
3. Architecture compliance pass (audit)
4. Severity tiers
5. Test levels
6. What to test per layer
7. Mock strategy
8. Per-runtime testing notes
9. CI gates

---

## 1. Fitness functions — structure (P11–P17)

A structural check walks the module tree and fails the build on violations. Wire it into CI and make failures blocking.

**What to enforce:**

| Check | Rule | Principle |
| --- | --- | --- |
| No technical-layer folders | Reject folders named `service/`, `controller/`, `entity/`, `repository/`, `dto/`, `types/`, `handlers/`, `models/` **inside** a module | P11, P12 |
| No single-file folders | A folder must hold ≥2 cohesive files. Exception: `__test__/` | P14 |
| Depth limit | ≤2 for flat modules, ≤3 for subdomain modules; `shared/` exempt | P13 |
| No README inside aggregates | README only at the package root | P17 |
| Aggregate size | Warn at ~15 files, fail (or flag strongly) at ~25 | P15 |
| Suffix naming | Files match `<aggregate>.<role>.<ext>` where role ∈ {entity, repository, service, controller, handler, resolver, dto, types, constants, …} | P12, P18 |

```javascript
// sketch — walk each module, collect violations, exit non-zero
const LAYER_FOLDERS = new Set(['service','services','controller','controllers','entity','entities',
  'repository','repositories','dto','dtos','types','model','models','handler','handlers','usecase','use-cases'])

function validateStructure(moduleRoot) {
  const violations = []
  for (const dir of walkDirs(moduleRoot)) {
    const name = basename(dir)
    const rel  = relative(moduleRoot, dir)
    if (LAYER_FOLDERS.has(name) && !rel.startsWith('shared')) {
      violations.push({ path: rel, issue: 'Technical-layer folder inside a module (P11)' })
    }
    const files = sourceFilesIn(dir)
    if (files.length === 1 && name !== '__test__') {
      violations.push({ path: rel, issue: 'Single-file folder — use a suffix instead (P14)' })
    }
    if (files.length > 25) {
      violations.push({ path: rel, issue: `Aggregate has ${files.length} files — strong split candidate (P15)` })
    }
    if (rel.split(sep).length > 3 && !rel.startsWith('shared')) {
      violations.push({ path: rel, issue: 'Depth > 3 (P13)' })
    }
    if (existsSync(join(dir, 'README.md')) && rel !== '') {
      violations.push({ path: rel, issue: 'README inside an aggregate (P17)' })
    }
  }
  return violations
}
```

---

## 2. Fitness functions — boundaries and state (P1, P3, P8)

| Check | Rule |
| --- | --- |
| No deep cross-module imports | A module may import another module only through its barrel/facade entry point — never a deep path into its internals |
| No duplicate entity names | Two modules must not declare the same entity/model name |
| No unprefixed entity names | Every persisted entity is prefixed with its module name |
| No cross-context relations | No foreign key or ORM relation crosses a module boundary |
| No shared mutable state | No exported mutable singleton crossing a boundary |
| No direct cross-module service calls | Cross-module interaction is a facade call or an event, never a reach into another module's service |
| Domain independence | Domain-layer files import nothing from infrastructure or transport |

The concrete grep-based detectors and a ready-to-use pre-merge gate script are in `state-isolation.md` §8–9. The domain-grouping and component-size fitness functions are in `decomposition-pipeline.md`.

**Treat structural failures as blocking. Treat boundary findings as blocking once the team has adopted the convention** — introducing them as warnings first, then flipping to blocking, is the practical migration path for an existing codebase.

---

## 3. Architecture compliance pass (audit)

Use this for **reviews or audits** without assuming any tooling. Treat items as **signals**, not proof — confirm with domain experts and by reading the code.

### Dependency and API signals

- **Inbound vs outbound:** dependencies should align with the chosen architecture (domain at the center, adapters outside). **Inward leaks** of infrastructure types into core logic are a smell.
- **Public surface:** can you list a module's exported operations/events/types **without** including storage or internal services? If not, the boundary is leaky.
- **Neighbor imports:** types or clients from module A used in module B — are they only **contract** types, or persistence/implementation types?

### Persistence and data signals

- **Reach-through:** references to another context's **physical** data (schema, table, collection, bucket name) outside an agreed contract.
- **Naming collisions:** the same logical name used for different things, or shared global ids with no documented mapping rule.
- **Transaction ownership:** writes that span contexts with no **saga**, **outbox**, or **single-owner** rule and no documented failure cases.

### Operational signals

- **Blame:** incidents where "we don't know which module owns this row/behavior" → an ownership or observability gap.
- **Cascades:** one dependency's slowdown or failure takes down unrelated user journeys → missing **timeouts**, **bulkheads**, or **degradation** paths.

### Reporting

For each finding give: the signal observed, the concrete evidence (file:line, table, import edge, incident), the principle violated, the severity tier, and a specific recommendation. No generic advice.

---

## 4. Severity tiers

| Tier | Meaning |
| --- | --- |
| **P0** | Data corruption risk, security boundary violation, or cross-context persistence with no contract |
| **P1** | Unclear ownership, leaky public API, missing failure semantics at boundaries |
| **P2** | Observability gaps, composability smells, tech debt that increases future coupling |

**Maturity note:** scoring is **qualitative** unless the team defines numeric gates. Use trends rather than absolute scores — fewer P0/P1 findings over time, clearer contracts, fewer cross-module import violations per release.

---

## 5. Test levels

| Level | What to test | Where it lives | Dependencies |
| --- | --- | --- | --- |
| **Domain** | Entity behavior, value-object validation, aggregate invariants | `<aggregate>/__test__/` | **None** — pure, no framework |
| **Application** | Business rules through services (or command/query handlers) | `<aggregate>/__test__/` | Mocked repository interfaces + mocked event publisher |
| **Transport** | Input parsing, delegation, response mapping, auth guards | next to the transport file | Mocked service |
| **Integration** | Wiring resolves, the module works within its own boundary | `<module>/__test__/` | The real module, a test database |
| **Contract** | The module's facade and its published event schemas | `<module>/__test__/` | Fakes at the ports |
| **End-to-end** | The full lifecycle through the real entry point | `__test__/e2e/<flow>` | The full app, a test database |

Unit tests live **next to the aggregate** in `__test__/`, not beside production files. E2E tests are centralized **per flow**, not per endpoint.

---

## 6. What to test per layer

**Domain — the most valuable tests you will write.** Construct the entity directly and assert its behavior and invariants. No mocks, no container, no database.

```ts
describe('BillingPlan', () => {
  it('is active when the price is positive', () => {
    expect(new BillingPlan('p-1', 'Pro', 2999, MONTHLY, new Date()).isActive()).toBe(true)
  })
  it('is not active when the price is zero', () => {
    expect(new BillingPlan('p-1', 'Free', 0, MONTHLY, new Date()).isActive()).toBe(false)
  })
  it('allows an upgrade to a higher-priced plan and rejects a downgrade', () => {
    const basic = new BillingPlan('p-1', 'Basic', 999, MONTHLY, new Date())
    const pro   = new BillingPlan('p-2', 'Pro',  2999, MONTHLY, new Date())
    expect(basic.canUpgradeTo(pro)).toBe(true)
    expect(pro.canUpgradeTo(basic)).toBe(false)
  })
})
```

**Application — mock the repository *interface*, never the persistence library.** Assert the business outcome, the persistence call, *and the event published*.

```ts
it('creates a billing plan and publishes the event', async () => {
  repository.save.mockImplementation(async plan => plan)

  const result = await service.create('Pro', 2999, MONTHLY)

  expect(result.name).toBe('Pro')
  expect(repository.save).toHaveBeenCalledTimes(1)
  expect(events.publish).toHaveBeenCalledWith(
    'billing.plan.created',
    expect.objectContaining({ planId: expect.any(String) }),
  )
})

it('throws a domain error when the plan is not found', async () => {
  repository.findById.mockResolvedValue(null)
  await expect(service.findById('nonexistent')).rejects.toThrow(BillingPlanNotFoundError)
})
```

**Transport — assert delegation and mapping, not business rules.** With a mocked service, verify the handler parsed the input, called the operation once with the right arguments, and mapped the result and errors correctly.

**Integration — assert the module wires up and works inside its own boundary.** Build the module's container/wiring and verify every provider resolves, the repository actually reads and writes, and no dependency is missing.

**Contract — assert the public surface has not drifted.** The facade signature, the DTO shapes, and the published event schemas are what other modules depend on. A test that pins them turns a silent breaking change into a red build.

**E2E — assert the full lifecycle through the real entry point**, including authentication, validation rejection, and status/exit codes:

```ts
it('creates a plan',           () => post('/billing/plans', valid).expect(201))
it('rejects invalid data',     () => post('/billing/plans', { name: '', priceInCents: -1 }).expect(400))
it('requires authentication',  () => get('/billing/plans').expect(401))
```

---

## 7. Mock strategy

**Mock at the port, never inside the module.** The things you mock should be exactly the interfaces your module declares: repositories, event publishers, and external-system ports. Everything else runs for real.

```ts
export function createMockRepository<T>() {
  return { findById: jest.fn(), findAll: jest.fn(), save: jest.fn(), delete: jest.fn() }
}
export function createMockEventPublisher() {
  return { publish: jest.fn() }
}
export function createMockService(methods: string[]) {
  return Object.fromEntries(methods.map(m => [m, jest.fn()]))
}
```

**Prefer in-memory fakes over mocks for ports you use heavily.** An `InMemoryBillingPlanRepository` that actually stores objects produces more realistic tests than a mock that returns whatever the test told it to — and it doubles as the local development adapter.

**Never mock:** the entity under test, value objects, or pure functions. If a test needs to mock a domain object, the design is wrong.

---

## 8. Per-runtime testing notes

| Runtime | Notes |
| --- | --- |
| **HTTP server** | E2E through the real HTTP stack with the validation pipeline enabled; assert status codes, not just bodies. |
| **Serverless** | Test the handler as a plain function with a constructed event object — no emulator needed. Keep the composition root injectable so tests can substitute adapters. Add one deployed smoke test per function. |
| **Queue worker** | Test the message handler as a function. **Always add an idempotency test**: deliver the same message twice and assert the end state is identical and side effects happened once. |
| **Scheduled job** | Test resumability: run, kill mid-way, run again, assert no duplicated work and correct completion. |
| **CLI** | Test the command function directly; assert exit codes and stderr separately from stdout. |

---

## 9. CI gates

Run in this order, failing fast:

1. **Type check / compile** with the strictest settings the language offers (no implicit `any`, null-strictness, unchecked-index checks).
2. **Lint / format** — one fast tool.
3. **Structure fitness function** (§1) — blocking.
4. **Boundary fitness function** (§2) — blocking once adopted.
5. **Unit tests** (domain + application) — coverage target ≥80%, with the domain layer closer to 100%.
6. **Integration tests** per module.
7. **Contract tests** — public surfaces and event schemas.
8. **E2E** on the main flows.

**Exit criteria before shipping a module:**

- [ ] No duplicate entity names across modules
- [ ] No direct cross-module internal imports
- [ ] Every module builds and tests independently
- [ ] Event contracts validated, consumers idempotent
- [ ] Each module owns its state; no cross-boundary foreign keys
- [ ] Every adapter has a documented resilience policy
- [ ] The build graph respects module boundaries
