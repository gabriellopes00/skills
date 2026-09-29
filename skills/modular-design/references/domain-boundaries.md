# Domain & Boundaries Reference (DDD Strategic Design)

Strategic design defines the boundaries — always applied, in any language or framework. Tactical design (rich models) fills them in proportion to each subdomain's value; see `module-internals.md` for that.

This is strategic analysis: focus on **WHAT** domains exist, not **HOW** to implement them.

## Table of Contents

1. Subdomain classification
2. Bounded contexts & ubiquitous language
3. Context integration patterns
4. Cohesion assessment and scoring
5. Detecting low-cohesion issues
6. Bounded context sizing
7. Creating a bounded context (workflow)
8. When to split or merge
9. Sub-units inside a bounded context
10. Analysis process for an existing codebase
11. Output formats
12. Worked examples
13. Anti-patterns and common mistakes
14. Signal words and key questions

---

## 1. Subdomain classification

Classify every subdomain — it tells you where to invest depth, who works on it, and how volatile it is.

- **Core** — competitive advantage, highest business value, most complex. Best people, full tactical depth. Indicators: complex business logic, frequent change, domain experts needed.
- **Supporting** — essential and business-specific, but not differentiating. Moderate depth. Indicators: supports the Core, moderate complexity, business-specific rules.
- **Generic** — a solved problem you consume (often wrapping an external service). Minimal depth. Indicators: well-understood problem, low differentiation, standard functionality, could be bought or outsourced.

**Decision tree:**

```
Is it a competitive advantage / does it differentiate the business?
├─ YES → CORE DOMAIN
└─ NO  → Is it business-specific? Does it require domain knowledge?
         ├─ YES → SUPPORTING SUBDOMAIN
         └─ NO  → GENERIC SUBDOMAIN
```

**Common examples:**

| Class | Typical members |
| --- | --- |
| **Generic** (can outsource) | Authentication/authorization, email/SMS sending, file storage, logging/monitoring, caching, basic search indexing, payment processing |
| **Supporting** (business-specific) | Inventory management, order fulfillment, content moderation, user notifications, reporting/analytics, invoice generation, shipping |
| **Core** (competitive advantage) | A unique recommendation algorithm, custom pricing strategy, proprietary matching, specialized risk assessment, a custom forecasting model |

**Volatility follows classification** — this matters for coupling analysis (`coupling-analysis.md`):

| Type | Volatility | Why |
| --- | --- | --- |
| Core | High | The area the business most wants to evolve |
| Supporting | Low | Simple CRUD, core support, little algorithmic complexity |
| Generic | Minimal | Auth, billing, email, logging, storage — stable, well-understood |

---

## 2. Bounded contexts & ubiquitous language

A **bounded context** is an explicit **linguistic** boundary where every term has one unambiguous meaning. It is not primarily a technical boundary.

- Aim for **1 subdomain ≈ 1 bounded context** (ideal, not a law).
- Inside the boundary, all ubiquitous-language terms are unambiguous.
- The same word meaning different things in two places is a real boundary signal, not an accident. ("Ticket" as a unit of work vs. a set of payments. "Customer" in Sales vs. Support.)
- Group by **business language**, never by technical layer.

**Bounded context detection:**

```
Same term, different meaning in two places?
├─ YES → DIFFERENT CONTEXTS
└─ NO  → SAME CONTEXT (but verify — identical usage everywhere is suspicious too)
```

**Clear boundary signs:** distinct ubiquitous language; concepts unambiguous inside; different meanings across contexts; clear integration points.

**Unclear boundary signs:** the same terms with the same meanings everywhere; concepts used identically system-wide; no linguistic differences; tight coupling everywhere.

---

## 3. Context integration patterns

Pick deliberately — the choice determines both coupling strength and failure semantics.

| Pattern | Use when | Shape |
| --- | --- | --- |
| **Anti-Corruption Layer (ACL)** | Protecting your model from an external/legacy/vendor model. Use for **every** external system. | Translation layer (port + adapter) — see `contracts-and-communication.md` |
| **Customer/Supplier** | Clear upstream/downstream, and upstream will consider downstream needs | API contract, negotiated |
| **Conformist** | Downstream conforms to upstream's model (no leverage to negotiate) | Accept the upstream model as-is |
| **Open Host Service** | You publish an interface others integrate against | REST/GraphQL/gRPC API |
| **Published Language** | A well-documented shared schema (e.g. domain events) | Versioned event/message schema |
| **Shared Kernel** | Rarely — small, stable, intentional shared model | Small shared value objects (Money, Address, Email). Never share entities. Creates coupling. |
| **Domain Events (Pub/Sub)** | Multiple consumers, eventual consistency acceptable | `OrderPlaced` → [Billing, Shipping, Analytics] |

**Integration timing:**

- **Synchronous** — use sparingly, when immediate consistency is required. Example: Order → Payment (needs an immediate response).
- **Asynchronous** — prefer, when eventual consistency is acceptable. Example: Order → Shipping.
- **Event-driven** — best for decoupling, when multiple contexts need to react. Example: `OrderPlaced` → Billing, Shipping, Analytics.

---

## 4. Cohesion assessment and scoring

**High cohesion (keep together):** shared vocabulary; used together; direct relationships; change together; solve the same business problem.

**Low cohesion (review the boundary):** mixed vocabularies; rarely used together; no relationship; changes don't propagate; solve different problems.

Some cross-context dependency is normal. Generic subdomains naturally have lower cohesion.

**Cohesion score (0–10):**

```
Score = Linguistic (0–3) + Usage (0–3) + Data (0–2) + Change (0–2)

Linguistic — same vocabulary?
  3 = all terms shared   2 = most shared   1 = some shared   0 = different vocabulary
Usage — used together?
  3 = always            2 = frequently    1 = sometimes     0 = rarely
Data — direct relationships?
  2 = direct entity relationships          1 = indirect      0 = none
Change — change together?
  2 = always            1 = sometimes                        0 = independently
```

**Interpretation:**

- **8–10 ✅ HIGH** — strong subdomain candidate; good bounded-context boundary.
- **5–7 ⚠️ MEDIUM** — review boundaries; may need refinement.
- **0–4 ❌ LOW** — likely the wrong grouping; needs separation.

Low cohesion does not always mean "bad" — it means "needs attention."

---

## 5. Detecting low-cohesion issues

Five rules to apply while scanning:

**Rule 1 — Linguistic Mismatch.** Different business vocabularies mixed in one unit. *Example:* `User` (identity) and `Subscription` (billing) in the same service. *Action:* separate into different bounded contexts.

**Rule 2 — Cross-Domain Dependencies.** Tight coupling between domains. *Example:* Service A directly instantiates entities from Domain B. *Action:* interface- or event-based integration.

**Rule 3 — Mixed Responsibilities.** A single class handles multiple business concerns. *Example:* a service handling both billing and content. *Action:* split by subdomain.

**Rule 4 — Generic in Core.** Generic functionality embedded in core business logic. *Example:* email sending inside the billing service. *Action:* extract to a Generic subdomain behind a port.

**Rule 5 — Unclear Boundaries.** You cannot determine which domain a concept belongs to. *Example:* an entity with relationships into multiple domains. *Action:* clarify boundaries; possibly split the concept.

---

## 6. Bounded context sizing

Size follows the ubiquitous language, not a file count. This is sometimes called the **Mozart Principle** — "neither too few notes nor too many": a context should contain exactly the concepts its language needs to be complete, and no others.

**Too small** — gaping holes in the language; incomplete business capability; too many integration points; fragments of concepts.

```
❌ ProductContext (only Product)
   InventoryContext (only Stock)
   PricingContext (only Price)
→ Should be: CatalogContext
```

**Just right** — complete ubiquitous language; a full business capability; clear integration points; cohesive concepts.

```
✅ CatalogContext
   ├── Product
   ├── Category
   ├── Inventory
   └── Pricing
```

**Too large** — multiple vocabularies mixed; multiple business capabilities; low internal cohesion; muddy boundaries.

```
❌ BusinessContext
   ├── Order    (order language)
   ├── Product  (catalog language)
   ├── User     (identity language)
   └── Payment  (billing language)
→ Should be: 4 separate contexts
```

**Rough guides** (never force uniformity): small domain 3–5 concepts if complete; medium 6–15; large 16+ if genuinely cohesive. At the *grouping* level, aim for **3–7 domains** overall; >10 suggests merging, <3 suggests splitting.

---

## 7. Creating a bounded context (workflow)

Use when introducing a **new** cohesive area (greenfield module or extracted domain).

1. **Scope and language** — Name the context; list its core nouns and verbs (the ubiquitous language). Reject vague names that collide with other contexts.
2. **Responsibilities** — What decisions happen **only** here? What is explicitly *out* of scope?
3. **State ownership** — Which facts are **authoritative** here? Where are they stored conceptually (even if the storage technology is undecided)?
4. **Public contract** — Which operations and/or events may other contexts use? Version or evolve this contract intentionally.
5. **Integrations** — For each neighbor: sync call, async message, shared read model, or batch sync? Document **consistency** (immediate/eventual) and **failure** behavior.
6. **Invariants and lifecycles** — What must always be true inside this boundary? What starts and completes a lifecycle?
7. **Isolation check** — Can you test core behavior **without** spinning up unrelated contexts (fakes at the ports)?
8. **Observability** — How will you trace a request or job through **this** context with clear identifiers?

While designing cross-module interaction: prefer the **minimal** contract; define **timeouts**, **retries**, and **idempotency** for async; never take "temporary" direct store access as a shortcut.

---

## 8. When to split or merge

**Default: fewer boundaries until real pain appears.** "Flat is often better" than premature fragmentation. Splitting adds coordination, versioning, and operational cost.

### Six-criteria test (favor a split when several are true)

| # | Criterion | Question |
| --- | --- | --- |
| 1 | **Language** | Do the sub-areas use **different vocabulary**, or conflicting definitions of the same word? |
| 2 | **Rate of change** | Do parts **change on different cadences**, or for unrelated reasons (most edits touch one side)? |
| 3 | **Scale / SLO** | Do parts need **different** throughput, latency, or availability targets? |
| 4 | **Consistency** | Do they need **different transaction boundaries** (cannot cleanly share one atomic write model)? |
| 5 | **Ownership** | Would **different teams** or clearer ownership lines reduce conflict and review churn? |
| 6 | **Pain signal** | Is there **observable** integration pain: ripple effects, fear of change, unclear bug ownership? |

Favor **high cohesion** inside a module and **low, explicit coupling** between modules. If the only motivation is "files got big" or folder aesthetics, **merge or wait**.

### When to merge, or not split yet

- Boundaries are **artificial** (same language, same lifecycle, constant cross-calls).
- Splitting would **duplicate** logic or data with no clear **single writer** rule.
- The team is not ready to **own** contracts, versioning, and operations for extra units.

### Decision prompts

- Would separation **reduce** accidental coupling more than it **increases** coordination cost?
- Is there a natural **ubiquitous language** boundary, or only a technical seam?

---

## 9. Sub-units inside a bounded context

Sometimes one outer boundary is right, but inside it there are named sub-areas (subdomains, feature areas). The principles still apply **within** the context.

**Ownership.** Each sub-unit should **own** its slice of model and persistence concerns. Avoid one mega registration layer that wires **every** store and repository for **every** sub-unit in one place — that encourages reach-through and hidden coupling.

**Cross-sub-unit access.** Prefer **internal application APIs** or **thin internal facades** (same context, explicit surface) over peers importing each other's storage types directly. For async flows inside the context, prefer **enriched payloads** so handlers do not chat across sub-units for data that could have travelled with the event.

**Shared kernel inside the context.** Small, stable shared types or enums can live in a **narrow** shared area — but resist a growing "utils" dump that becomes the real coupling point.

**Anti-pattern.** A single "persistence" or "data" sub-module that is the **only** place knowing about all tables for all sub-units, with everyone reaching through it — the same problems as cross-context reach-through, *inside* the boundary.

---

## 10. Analysis process for an existing codebase

### Phase 1 — Extract concepts

Scan for **business** concepts (not infrastructure):

1. **Entities** — domain models with identity. Patterns: entity annotations, domain model classes. Focus on business concepts, not technical classes.
2. **Services** — business operations. Patterns: `*Service`, `*Manager`, `*Handler`. Focus on business logic, not technical utilities.
3. **Use cases** — business workflows. Patterns: `*UseCase`, `*Command`, `*Handler`. Focus on business processes, not CRUD.
4. **Entry points** — controllers, resolvers, queue handlers, CLI commands, API endpoints. Focus on business capabilities, not technical routes.

### Phase 2 — Group by ubiquitous language

For each concept determine its **primary language context** (`Subscription`, `Invoice`, `Payment` → Billing language; `Movie`, `Video`, `Episode` → Content language; `User`, `Authentication` → Identity language), find where **term meanings change**, and note which concepts naturally belong together.

### Phase 3 — Identify subdomains

A subdomain has: a distinct business capability; independent business value; a unique vocabulary; multiple related entities working together; a cohesive set of business operations. Then classify each with the decision tree from §1.

### Phase 4 — Assess cohesion

Score each candidate grouping with §4. Flag anything below 8.

### Phase 5 — Detect low-cohesion issues

Apply the five rules from §5.

### Phase 6 — Map bounded contexts

For each subdomain, propose a bounded context: a name reflecting the ubiquitous language, a complete domain model, explicit integration points, and a clear linguistic boundary. Choose an integration pattern per neighbor (§3).

---

## 11. Output formats

### Domain map (per domain/subdomain)

```markdown
## Domain: {Name}

**Type**: Core Domain | Supporting Subdomain | Generic Subdomain
**Ubiquitous Language**: {key business terms}
**Business Capability**: {what business problem it solves}

**Key Concepts**:
- {Concept} (Entity | Service | UseCase) — {brief description}

**Subdomains** (if applicable):
1. {Subdomain} (Core | Supporting | Generic)
   - Concepts: {list}
   - Cohesion: {score}/10
   - Dependencies: → {other domains}

**Suggested Bounded Context**: {Name}Context
- Linguistic boundary: {where terms have specific meaning}
- Integration: {how it integrates with other contexts}

**Dependencies**:
- → {OtherDomain} via {interface / event / API}
- ← {OtherDomain} via {interface / event / API}

**Cohesion Score**: {score}/10
```

### Cohesion matrix

```markdown
| Domain A | Domain B | Cohesion | Issue              | Recommendation          |
| -------- | -------- | -------- | ------------------ | ----------------------- |
| Billing  | Identity | 2/10     | ❌ Direct coupling | Use interface           |
| Content  | Billing  | 6/10     | ⚠️ Usage tracking  | Event-based integration |
```

### Low-cohesion report

```markdown
### Priority: High

**Issue**: {description}
- **Location**: {file / class / method}
- **Problem**: {what's wrong}
- **Concepts**: {involved concepts}
- **Cohesion**: {score}/10
- **Recommendation**: {suggested fix}
```

### Bounded context map

```markdown
### {ContextName}Context

**Contains Subdomains**:
- {Subdomain1} (Core)
- {Subdomain2} (Supporting)

**Ubiquitous Language**:
- Term: definition *in this context*

**Integration Requirements**:
- Consumes from: {OtherContext} via {pattern}
- Publishes to: {OtherContext} via {pattern}

**Implementation Notes**:
- Separate persistence ownership
- Independent deployability (potential)
- Explicit API boundaries
```

---

## 12. Worked examples

### Example A — E-commerce platform

**Language groups:** Catalog (Product, Category, SKU, Price) · Inventory (Stock, Warehouse, Reservation) · Order (Order, Cart, Checkout) · Payment (Charge, Refund, Gateway) · Shipping (Shipment, Carrier, Tracking).

| Subdomain | Class | Rationale |
| --- | --- | --- |
| Product Catalog | Core | Merchandising and discovery differentiate the store |
| Order Processing | Core | The transactional heart |
| Inventory Management | Supporting | Essential, business-specific, not differentiating |
| Shipping | Supporting | Business-specific rules over a generic capability |
| Payment Processing | Generic | Consume a gateway behind an ACL |

Low-cohesion issue found: `User` from Identity imported directly into Order → replace with a `CustomerId` value object.

### Example B — Healthcare system

Patient Care (Core) · Appointment Management (Supporting) · Pharmacy (Supporting) · Medical Billing (Supporting).

**Key insight — "Patient" means three different things:**

| Context | "Patient" means | Properties it cares about |
| --- | --- | --- |
| Clinical | A medical subject | Diagnosis, vitals, allergies |
| Scheduling | An appointment holder | Availability, preferences |
| Billing | A payer / beneficiary | Insurance, balance, claims |

→ These are **different bounded contexts** despite sharing the word "Patient". Translate at the boundary rather than sharing one `Patient` model.

### Example C — SaaS project management tool

Project Management (Core) · Collaboration (Core/Supporting — depends on whether it differentiates) · Identity & Access (Generic) · Billing (Supporting) · Notifications (Generic).

**Low-cohesion issue — the `User` entity used everywhere:**

```typescript
// ❌ Identity's concept leaked into every domain
class Project      { owner: User; members: User[] }
class Comment      { author: User }
class Subscription { subscriber: User }

// ✅ Context-specific identifiers instead
class Project      { ownerId: OwnerId; members: MemberId[] }
class Comment      { authorId: ParticipantId }
class Subscription { subscriberId: CustomerId }
```

### Example D — Streaming video platform

Content Catalog (Supporting) · Video Streaming (Supporting) · User Engagement (Supporting) · **Recommendation Engine (Core** — the proprietary algorithm is the competitive advantage) · Video Processing/Transcoding (Generic — buy it).

**Integration pattern — event-driven, so Recommendation never reads anyone's store:**

```
Catalog Context    ──publishes──▶ ContentPublished ──▶ Recommendation Context (consumes)
Engagement Context ──publishes──▶ UserWatched      ──▶ Recommendation Context (consumes)
```

### Patterns across all examples

**Pattern 1 — Identity leakage.** *Problem:* User/Identity entities used directly everywhere. *Solution:* context-specific identifiers — `OwnerId`/`MemberId` in Project, `CustomerId`/`SubscriberId` in Billing, `CreatorId`/`ViewerId` in Content.

**Pattern 2 — Shared kernel overuse.** *Problem:* large shared models used everywhere. *Solution:* a minimal shared kernel, mostly value objects. Share `UserId` (as a string/UUID) and `Email` (as a value object). Do not share the `User` or `Customer` entity.

**Pattern 3 — Core vs Supporting confusion.** Ask "is this our competitive advantage?" If yes → Core (best team, most attention). If no but business-specific → Supporting. If no and standard → Generic.

**Pattern 4 — Bounded context size.** See §6.

**Pattern 5 — Integration types.** See §3.

---

## 13. Anti-patterns and common mistakes

**Big Ball of Mud** — everything connected to everything, no clear boundaries, mixed vocabularies. *Prevention:* explicit bounded contexts.

**All-Inclusive Model** — a single model for the entire business; impossible global definitions; conflicts. *Prevention:* embrace multiple contexts.

**Mixed Linguistic Concepts** — different vocabularies in one context (e.g. User/Permission alongside Forum/Post). *Prevention:* keep linguistic associations intact.

**Mistake 1 — Grouping by technical layer.**

```
❌ ControllerContext, ServiceContext, RepositoryContext
✅ OrderContext (all layers for orders), ProductContext (all layers for products)
```

**Mistake 2 — Sharing entities directly.**

```
❌ class Order { user: User }          // full entity from Identity
✅ class Order { customerId: CustomerId }  // value object
```

**Mistake 3 — One size fits all.** Do not force every domain to the same size. Size follows the ubiquitous language.

**Mistake 4 — Technical boundaries.** Contexts drawn for frontend-vs-backend, one-microservice-per-entity, or one-context-per-database are technical seams, not linguistic boundaries.

**Coupling red flags:**

```
❌ Direct entity import across domains:  import { User } from '@identity/entities'
❌ Service dependency across domains:    constructor(subscriptionService: SubscriptionService)
❌ Shared database tables across domains: FOREIGN KEY(user_id) REFERENCES users(id)

✅ Interface-based integration:  constructor(billingApi: BillingApiPort)
✅ Event-based communication:    eventBus.publish(new OrderPlaced(...))
✅ Value object sharing:         class Order { customerId: CustomerId }
```

**Do's ✅** — focus on business language, not code structure; let the ubiquitous language guide boundaries; measure cohesion objectively; identify clear integration points; classify every subdomain; look for linguistic boundaries first; collaborate with business stakeholders; use business language in domain names.

**Don'ts ❌** — don't group by technical layer; don't force a single global model; don't ignore linguistic differences; don't couple domains directly; don't create contexts from the architecture or the org chart; don't try to eliminate all dependencies (some are necessary); don't skip stakeholder validation.

---

## 14. Signal words and key questions

**Core signals:** "competitive advantage" · "unique to our business" · "our secret sauce" · "what makes us different" · "complex business rules" · "needs domain experts".

**Supporting signals:** "necessary but standard" · "business-specific" · "supports core operations" · "moderate complexity" · "internal tool".

**Generic signals:** "could buy this" · "standard functionality" · "well-known solution" · "common to all businesses" · "infrastructure".

**Low-cohesion signals:** "mixed concerns" · "different vocabularies" · "unrelated concepts" · "tight coupling" · "unclear boundary" · "linguistic mismatch".

**For subdomain classification:** Does this provide competitive advantage? Is it business-specific or generic? Is it essential to the core business? Could we outsource it? How often does it change? Does it require domain experts?

**For bounded-context definition:** Does this term have a different meaning elsewhere? Can we define all terms unambiguously here? Is this a complete business capability? Are all concepts linguistically related? Where do we translate between contexts? What are the integration points?

**For cohesion assessment:** Do these concepts share vocabulary? Are they used together frequently? Do changes affect them together? Do they solve the same business problem? Are they in the same lifecycle? Do they have direct relationships?

---

## Validation criteria

Good domain identification has:

- ✅ Clear boundaries with a distinct ubiquitous language
- ✅ High internal cohesion within domains
- ✅ Explicit, documented cross-domain dependencies
- ✅ Business alignment with real capabilities
- ✅ Actionable recommendations for the issues found

Always validate with domain experts when possible. This is strategic analysis — some cross-domain dependencies are normal, and Generic subdomains naturally score lower on cohesion.
