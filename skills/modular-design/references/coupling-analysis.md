# Coupling Analysis Reference

Analyze coupling with the **three-dimensional model** from *Balancing Coupling in Software Design* (Vlad Khononov):

1. **Integration Strength** — *what* is shared between components
2. **Distance** — *where* the coupling physically lives
3. **Volatility** — *how often* the components change

The guiding balance formula:

```
BALANCE = (STRENGTH XOR DISTANCE) OR NOT VOLATILITY
```

A design is **balanced** when:

- Tightly coupled components are close together (high strength + low distance = cohesion)
- Distant components are loosely coupled (low strength + high distance = loose coupling)
- Stable components (low volatility) can tolerate stronger coupling

This is Pattern 4 of the decomposition pipeline, and also stands alone as an audit.

## Table of Contents

1. When to use
2. Phase 1 — Context gathering
3. Phase 2 — Structural mapping and distance
4. Phase 3 — Integration strength
5. Phase 4 — Volatility
6. Phase 5 — Balance score
7. Phase 6 — The report
8. Quick reference: pattern → strength
9. Quick heuristics
10. Known limitations

---

## 1. When to use

- The user asks to "analyze coupling", "evaluate the architecture", or "check dependencies".
- You need to know whether a module should be extracted or merged.
- Changes in one module cascade to others unexpectedly, and nobody can say why.
- Concepts like connascence, cohesion, or Khononov's model come up.
- As step 4 of the decomposition pipeline, after inventory/duplication/flattening.

---

## 2. Phase 1 — Context gathering

**1.1 Scope.** Full codebase or one area? Primary level of abstraction — methods, classes, modules/packages, or services? Is git history available (needed to estimate volatility)?

**1.2 Business context** — ask the user or infer from the code:

- Which parts are the business **core** (competitive differentiator)?
- Which are infrastructure/generic support (auth, billing, logging)?
- What changes most frequently, according to the team?

This lets you classify subdomains, which drives volatility:

| Type | Volatility | Indicators |
| --- | --- | --- |
| **Core subdomain** | High | Proprietary logic, competitive advantage, the area the business most wants to evolve |
| **Supporting subdomain** | Low | Simple CRUD, core support, no algorithmic complexity |
| **Generic subdomain** | Minimal | Auth, billing, email, logging, storage |

(Full classification guidance: `domain-boundaries.md`.)

---

## 3. Phase 2 — Structural mapping and distance

**2.1 Module inventory.** For each module record: name and location (namespace/package/path), primary responsibility, and declared dependencies (imports, DI registrations, HTTP calls, queue subscriptions, shared tables).

**2.2 Dependency graph.** Build a directed graph: nodes = modules, edges = dependencies (`A → B` means "A depends on B").

> Note: the flow of **knowledge** is *opposite* to the dependency arrow. If `A → B`, then B is **upstream** and exposes knowledge to A (downstream).

**2.3 Distance.** Use the encapsulation hierarchy — the nearest common ancestor determines distance:

| Common ancestor level | Distance | Example |
| --- | --- | --- |
| Same method/function | Minimal | Two lines in the same method |
| Same object/class | Very low | Methods on the same object |
| Same namespace/package | Low | Classes in the same package |
| Same library/module | Medium | Libraries in the same project |
| Different services | High | Distinct services/deployables |
| Different systems/orgs | Maximum | External APIs, different companies |

**Social factor:** if two modules are maintained by **different teams**, increase the estimated distance by one level (Conway's Law).

---

## 4. Phase 3 — Integration strength

For each dependency, classify the strength — strongest (worst) to weakest (best).

### INTRUSIVE COUPLING — strongest, avoid

Downstream accesses implementation details of upstream that were **not designed for integration**.

**Code signals:**

- Reflection used to reach private members
- A service reading another service's database directly
- Dependence on the internal file or config structure of another module
- Monkey-patching internals
- Direct access to internal fields with no accessor

**Effect:** any internal change to upstream breaks downstream — even when the public interface is untouched. Upstream does not know it is being observed.

### FUNCTIONAL COUPLING — second strongest

Modules implement interrelated functionality: shared business logic, interdependent rules, or coupled workflows.

**Three degrees (weakest to strongest):**

**a) Sequential (temporal)** — modules must execute in a specific order.

```python
connection.open()    # must come first
connection.query()   # depends on open
connection.close()   # must come last
```

**b) Transactional** — operations must succeed or fail together.

```python
with transaction:
    service_a.update(data)
    service_b.update(data)   # both must succeed
```

**c) Symmetric — strongest** — the same business logic duplicated in multiple modules.

```python
# Module A
def is_premium_customer(c): return c.purchases > 1000

# Module B — duplicated rule! Must be kept in sync manually
def qualifies_for_discount(c): return c.purchases > 1000
```

> Symmetric coupling does **not** require the modules to reference each other. They can be fully independent in code and still carry this coupling. That is what makes it dangerous — and invisible to static analysis.

**General signals of functional coupling:** comments like "remember to update X when changing Y"; cascading test failures when one business rule changes; duplicated validation logic; needing to deploy several services simultaneously for one feature.

### MODEL COUPLING — third level

Upstream exposes its **internal domain model** as part of its public interface. Downstream knows and uses objects representing upstream's internal model.

```python
# Analysis module uses CRM's internal model directly
from crm.models import Customer

class Analysis:
    def process(self, customer_id):
        customer = crm_repo.get(customer_id)   # returns the FULL internal Customer
        status = customer.status               # only needs status, but knows everything
```

```typescript
// Service B consuming Service A's internal model over an API
interface CustomerFromServiceA {
  internalAccountCode: string   // internal detail leaked
  legacyId: number              // unnecessary internal field
  // ... many fields Service B does not need
}
```

**Degrees (via static connascence), weakest to strongest:**

- *Connascence of name* — knows the field names of the model
- *Connascence of type* — knows the specific types
- *Connascence of meaning* — interprets specific values (magic numbers, internal enums)
- *Connascence of algorithm* — must use the same algorithm to interpret the data
- *Connascence of position* — depends on element order (tuples, unnamed arrays)

### CONTRACT COUPLING — weakest, ideal

Upstream exposes an **integration-specific model** (a contract) separate from its internal model. The contract abstracts implementation details away.

```python
class CustomerSnapshot:   # integration DTO, not the internal model
    """Public integration contract — stable and intentional."""
    id: str
    status: str   # internal enum converted to a string
    tier: str     # only what consumers actually need

    @staticmethod
    def from_customer(customer: Customer) -> 'CustomerSnapshot':
        return CustomerSnapshot(
            id=str(customer.id),
            status=customer.status.value,
            tier=customer.loyalty_tier.display_name,
        )
```

**Characteristics of good contract coupling:** dedicated DTOs/view models per use case (not the domain model) · versionable contracts (V1, V2) · primitive or simple value types · explicit contract documentation (OpenAPI, Protobuf, JSON Schema, AsyncAPI) · the patterns Facade, Adapter, Anti-Corruption Layer, Published Language.

---

## 5. Phase 4 — Volatility

**4.1 Subdomain type** (preferred) — use the table in Phase 1.

**4.2 Git analysis** (when available):

```bash
# Commits per file in the last 6 months — change frequency
git log --since="6 months ago" --format="" --name-only | sort | uniq -c | sort -rn | head -20

# Files that change together frequently (temporal coupling)
# High co-change between modules = possible undeclared functional coupling
git log --since="6 months ago" --format="COMMIT" --name-only \
  | awk '/^COMMIT/{if(n>1)print s; s="";n=0;next}{s=s" "$0;n++}END{if(n>1)print s}' \
  | tr ' ' '\n' | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```

**4.3 Code signals:** many TODO/FIXME markers → area under active evolution (higher volatility) · many API versions (V1, V2, V3) → frequently changing area · fragile tests that break constantly → volatile area · "business rule: ..." comments → business logic, probably core.

**4.4 Inferred volatility.** Even a Supporting-subdomain module can have high volatility if it has Intrusive or Functional coupling with Core modules — changes in the core propagate into it.

---

## 6. Phase 5 — Balance score

**Simplified scale (0 = low, 1 = high):**

| Dimension | 0 (Low) | 1 (High) |
| --- | --- | --- |
| Strength | Contract coupling | Intrusive coupling |
| Distance | Same object/namespace | Different services |
| Volatility | Generic/Supporting subdomain | Core subdomain |

**Maintenance effort:**

```
MAINTENANCE_EFFORT = STRENGTH × DISTANCE × VOLATILITY
```

A 0 in any dimension means low effort — which is why the practical levers are: reduce strength (introduce a contract), reduce distance (move them together), or reduce volatility (stabilize the rule).

**Diagnosis table:**

| Strength | Distance | Volatility | Diagnosis |
| --- | --- | --- | --- |
| High | High | High | 🔴 **CRITICAL** — global complexity + high change cost |
| High | High | Low | 🟡 **ACCEPTABLE** — strong but stable (e.g. a legacy integration) |
| High | Low | High | 🟢 **GOOD** — high cohesion (they change together, they live together) |
| High | Low | Low | 🟢 **GOOD** — strong but static |
| Low | High | High | 🟢 **GOOD** — loose coupling (separate and independent) |
| Low | High | Low | 🟢 **GOOD** — loose coupling and stable |
| Low | Low | High | 🟠 **ATTENTION** — local complexity (unrelated components mixed together) |
| Low | Low | Low | 🟡 **ACCEPTABLE** — may generate noise, but low cost |

---

## 7. Phase 6 — The report

### 6.1 Executive summary

```
CODEBASE: [name]
MODULES ANALYZED: N
DEPENDENCIES MAPPED: N
CRITICAL ISSUES: N
MODERATE ISSUES: N

OVERALL HEALTH SCORE: [Healthy / Attention / Critical]
```

### 6.2 Dependency map

```
[ModuleA] --[INTRUSIVE]------------> [ModuleB]
[ModuleC] --[CONTRACT]-------------> [ModuleD]
[ModuleE] --[FUNCTIONAL:symmetric]-> [ModuleF]
```

### 6.3 Issues, by severity

```
ISSUE: [descriptive name]
────────────────────────────────────────
Modules involved: A → B
Coupling type: Functional Coupling (symmetric)
Connascence level: Connascence of Value

Evidence in code:
  [snippet or file:line of the found pattern]

Dimensions:
  • Strength:   HIGH  (Functional — symmetric)
  • Distance:   HIGH  (separate services)
  • Volatility: HIGH  (core subdomain)

Balance Score: CRITICAL 🔴
Maintenance: High — frequent changes propagate over a long distance

Impact: Any change to business rule [X] requires a simultaneous update in
        [A] and [B], which belong to different teams.

Recommendation:
  → Extract the shared logic into a dedicated module both can reference
    (DRY + contract coupling)
  → Or: accept the duplication and explicitly document the coupling
    (if volatility is lower than it appears)
```

### 6.4 Positive patterns found

```
✅ [ModuleX] uses dedicated integration DTOs — contract coupling done well
✅ [ServiceY] exposes only what consumers need — minimal model coupling
✅ [PackageZ] encapsulates its internal model — low implementation leakage
```

### 6.5 Prioritized recommendations

**High priority** (high impact, blocking evolution) → **Medium** (improves architectural health) → **Low** (incremental).

---

## 8. Quick reference: pattern → integration strength

| Pattern found | Integration strength | Action |
| --- | --- | --- |
| Reflection to access private members | Intrusive | Refactor urgently |
| Reading another service's database | Intrusive | Refactor urgently |
| Duplicated business logic | Functional (symmetric) | Extract to a shared module |
| Distributed transaction / saga | Functional (transactional) | Evaluate whether cohesion would be better |
| Mandatory execution order | Functional (sequential) | Document the protocol, or encapsulate it |
| Rich domain object returned across a boundary | Model coupling | Create an integration DTO |
| Internal enum shared externally | Model coupling | Create a public contract enum |
| Use-case-specific DTO | Contract coupling | ✅ Correct pattern |
| Versioned public interface/protocol | Contract coupling | ✅ Correct pattern |
| Anti-Corruption Layer | Contract coupling | ✅ Correct pattern |

---

## 9. Quick heuristics

**For integration strength:**

- "If I change an internal detail of module X, how many other modules need to change?"
- "Was this integration contract designed to be public, or is it accidental?"
- "Is there duplicated business logic that must be synchronized manually?"

**For distance:**

- "What is the cost of making a change that affects both modules?"
- "Do the teams maintaining these modules need to coordinate deployments?"
- "If one module fails, does the other stop working?"

**For volatility:**

- "Does this module encapsulate a competitive business advantage?"
- "Does the business team frequently request changes here?"
- "Is there a history of many refactors in this area?"

**For balance:**

- "Do components that need to change together live together in the code?"
- "Are independent components well separated?"
- "Where is there strong coupling with volatile *and* distant components?" → **that is the main problem.**

---

## 10. Known limitations

- **Volatility** is best estimated from real git history rather than static analysis alone.
- **Symmetric functional coupling** requires semantic reading of the code — static analysis tools generally do not detect it.
- **Organizational distance** (different teams) requires user input; it is not in the code.
- **Dynamic connascence** (timing, value, identity) is hard to detect without runtime observation.
- Analysis is a **starting point** — business context always refines the conclusions.

---

Based on *Balancing Coupling in Software Design* by Vlad Khononov (Addison-Wesley), with connascence terminology from Page-Jones.
