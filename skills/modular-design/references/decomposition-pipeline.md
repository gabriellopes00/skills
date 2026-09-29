# Decomposition Pipeline Reference (Patterns 1–5)

The ordered analysis pipeline to run **before** extracting anything from a monolith. Each pattern feeds the next.

**Run 1 → 2 → 3 → 4 → 5 in order.** Do not skip a step unless the user explicitly narrows scope — and if they do, state which patterns were skipped and how that limits the conclusions. Carry outputs forward: the inventory from Pattern 1 informs coupling in 4 and grouping in 5. Always cite concrete evidence (files, dependencies, metrics), never generic advice.

| Step | Pattern | Section |
| --- | --- | --- |
| 1 | Identify and size components | §1 |
| 2 | Common domain detection (duplication) | §2 |
| 3 | Flattening / hierarchy | §3 |
| 4 | Coupling analysis | **`coupling-analysis.md`** |
| 5 | Domain identification and grouping | §4 |

**Pattern 6 (create domain services / extraction) is out of scope.** After Pattern 5, phased extraction order, milestones, and migration planning is a separate deliverable. Do not improvise it inside this pipeline.

**Language grounding.** Structure alone produces *solution-space* groupings. Read `domain-boundaries.md` before or alongside Pattern 5 to validate boundaries against business language rather than folder structure.

---

## Vocabulary used throughout

- **Component** — an architectural building block: a **leaf node** in the directory/namespace structure containing source files, with a well-defined role and responsibility.
- **Subdomain / root namespace** — a parent namespace that has been *extended* by a child namespace. It is **not** a component.
- **Orphaned class** — a source file sitting in a root namespace (a non-leaf node). It belongs to no definable component.
- **Domain** — a logical grouping of components representing a distinct business capability.
- **CA (afferent coupling)** — the number of components that depend on a given component.

**Key rule:** components exist **only as leaf nodes**. If `services/billing` is extended to `services/billing/payment`, then `services/billing` becomes a subdomain, not a component.

---

## 1. Pattern 1 — Identify and size components

Identify the logical building blocks and calculate size metrics to assess decomposition feasibility and find oversized components.

### What users ask for

"Identify and size all components in this codebase" · "Which components are too large?" · "Create a component inventory for decomposition planning" · "Analyze component size distribution"

### Phase 1 — Identify components

1. **Map the directory/namespace structure.** Node/JS: `services/`, `routes/`, `models/`, `utils/`, `middleware/`. Java: package structure (`com.company.domain.service`). Python: module paths (`app/billing/payment`). Go: package directories. Serverless: one function or function-group per component.
2. **Identify leaf nodes.** Components are the deepest directories containing source files. `services/BillingService/` is a component; if `services/BillingService/payment/` exists, then `BillingService` is a subdomain.
3. **Create the inventory** — each component with its namespace/path, and any parent namespaces noted as subdomains.

```
services/
├── BillingService/          ← Component (leaf node)
│   ├── index.js
│   └── BillingService.js
├── CustomerService/         ← Component (leaf node)
│   └── CustomerService.js
└── NotificationService/     ← Component (leaf node)
    └── NotificationService.js
```

```
com.company.billing.payment   ← Component (leaf package)
com.company.billing.history   ← Component (leaf package)
com.company.billing           ← Subdomain (parent of payment/history)
```

### Phase 2 — Calculate size metrics

Use **statements**, not lines of code — statements account for complexity rather than formatting.

| Metric | How | Purpose |
| --- | --- | --- |
| **Statements** | Count executable statements terminated by `;` or newline. Include assignments, calls, returns, conditionals, loops. Exclude comments, blank lines, docstrings, and bare declarations. | Accurate size measure |
| **Files** | Count source files; exclude tests, configs, generated code, documentation | Complexity indicator |
| **Percent** | `(component_statements / total_statements) * 100` | Relative size |
| **Std dev** | `sqrt(sum((size - mean)^2) / (n - 1))`; deviation = `(size - mean) / stdDev` | Outlier detection |

Language notes: **JS/TS** — statements end in `;` or newline. **Java/C#** — statements end in `;`; exclude class/interface declarations. **Python** — executable statements only; exclude docstrings. **Go** — statements per line; exclude type declarations.

### Phase 3 — Identify size issues

**Oversized** (split candidates): exceeds the app-size threshold · more than **2 standard deviations** above the mean · contains multiple distinct functional areas.

| App size | Oversized threshold |
| --- | --- |
| Small (<10 components) | >30% of codebase |
| Medium (10–20 components) | >15% of codebase |
| Large (>20 components) | >10% of codebase |

**Undersized** (consolidation candidates): under 1% of the codebase · more than 1 std dev below the mean · only a few files with minimal functionality.

**Well-sized:** within 1–2 std dev of the mean; a single cohesive functional area.

> Standard deviation is more reliable than a fixed percentage — always compute both.

### Output

```markdown
## Component Inventory

| Component Name  | Namespace/Path               | Statements | Files | Percent | Status       |
| --------------- | ---------------------------- | ---------- | ----- | ------- | ------------ |
| Billing Payment | services/BillingService      | 4,312      | 23    | 5%      | ✅ OK        |
| Reporting       | services/ReportingService    | 27,765     | 162   | 33%     | ⚠️ Too Large |
| Notification    | services/NotificationService | 1,433      | 7     | 2%      | ✅ OK        |

## Size Analysis Summary

**Total Components**: 18
**Total Statements**: 82,931
**Mean Component Size**: 4,607 statements
**Standard Deviation**: 5,234 statements

**Oversized** (>2 std dev or >threshold):
- Reporting (33% — 27,765 statements) — consider splitting into:
  Ticket Reports · Expert Reports · Financial Reports

**Undersized** (<1 std dev):
- Login (2% — 1,865 statements) — consider consolidating with Authentication
```

Status legend: ✅ **OK** (within 1–2 std dev) · ⚠️ **Too Large** (>threshold or >2 std dev above mean) · 🔍 **Too Small** (<1% or >1 std dev below mean).

A size distribution bar chart helps: `████████████████ 33% (Reporting)` / `████ 9% (Ticket Assign)` / …

Recommendations should be prioritized: **High** — split oversized components (with the proposed split and expected size of each part). **Medium** — review undersized components for consolidation. **Low** — monitor well-sized components during decomposition.

### Fitness functions

```javascript
// Alert if any component exceeds a share of the codebase
function checkComponentSize(components, threshold = 0.1) {
  const total = components.reduce((s, c) => s + c.statements, 0)
  return components
    .filter(c => c.statements / total > threshold)
    .map(c => ({ component: c.name, percent: ((c.statements / total) * 100).toFixed(1), issue: 'Exceeds size threshold' }))
}

// Alert if a component is more than 2 standard deviations from the mean
function checkStandardDeviation(components) {
  const sizes = components.map(c => c.statements)
  const mean = sizes.reduce((a, b) => a + b, 0) / sizes.length
  const stdDev = Math.sqrt(sizes.reduce((s, x) => s + (x - mean) ** 2, 0) / (sizes.length - 1))
  return components
    .filter(c => Math.abs(c.statements - mean) > 2 * stdDev)
    .map(c => ({ component: c.name, deviation: ((c.statements - mean) / stdDev).toFixed(2), issue: 'More than 2 std dev from mean' }))
}
```

### Do's / Don'ts

**Do ✅** use statements not lines · identify components as leaf nodes only · calculate both percentage and standard deviation · consider app size when setting thresholds · document the path for each component · produce a visual size distribution.

**Don't ❌** count test files in component size · treat parent directories as components · use fixed thresholds regardless of app size · ignore small components · skip the standard-deviation calculation · mix infrastructure and domain components in one analysis.

---

## 2. Pattern 2 — Common domain detection

Find domain functionality duplicated across components and evaluate consolidation — **with the coupling impact computed before recommending anything.**

### Domain vs infrastructure

| Type | Description | Examples | Consolidate here? |
| --- | --- | --- | --- |
| **Domain** | Business processing, common to **some** processes | Notification, audit, validation, formatting | ✅ Yes |
| **Infrastructure** | Operational concerns, common to **all** processes | Logging, metrics, security, DB connections | ❌ No (handled separately) |

### Phase 1 — Common namespace patterns

Extract the **leaf node** name from every component namespace (`services/billing/notification` → `notification`) and group by it.

```markdown
**Notification Components**: services/customer/notification · services/ticket/notification · services/survey/notification
**Audit Components**: services/billing/audit · services/ticket/audit · services/survey/audit
```

Filter out infrastructure patterns (`.util`, `.helper`, `.common`). Focus on `.notification` / `.notify` / `.email`, `.audit` / `.auditing` / `.log`, `.validation` / `.validate` / `.validator`, `.format` / `.formatter`, `.report` / `.reporting` (when the functionality is genuinely similar).

### Phase 2 — Shared classes

Scan imports/dependencies and find classes used by **2+ components**; classify each as domain or infrastructure.

```markdown
**Domain Classes**: SMTPConnection — used by 5 components · AuditLogger — used by 8 · DataFormatter — used by 3
**Infrastructure Classes** (exclude): Logger — used by all · Config — used by all
```

```javascript
function extractLeafNode(ns) { const p = ns.split('/'); return p[p.length - 1] }

function groupByLeafNode(components) {
  const groups = {}
  for (const c of components) (groups[extractLeafNode(c.namespace)] ??= []).push(c)
  return groups
}

function findSharedClasses(components) {
  const usage = {}
  for (const c of components) for (const imp of c.imports) (usage[imp] ??= []).push(c.name)
  return Object.entries(usage).filter(([, users]) => users.length > 1).map(([cls, usedBy]) => ({ class: cls, usedBy }))
}
```

### Phase 3 — Functionality similarity

Read the source of each candidate. Identify what each does, note similarities and differences, and judge feasibility: are the differences **minor and configurable**? Can they be **abstracted**? Is the functionality **truly** the same?

```markdown
**Notification Components**
- CustomerNotification: sends billing notifications
- TicketNotification: sends ticket assignment notifications
- SurveyNotification: sends survey emails

Similarities: all send emails to customers
Differences: email content/templates, triggers
Consolidation Feasibility: ✅ High — differences are in content, not mechanism; abstract with templates + context
```

### Phase 4 — Coupling impact (do not skip)

```markdown
**Before Consolidation**
- CustomerNotification: CA = 2
- TicketNotification:  CA = 2
- SurveyNotification:  CA = 1
- Total CA: 5

**After Consolidation**
- Notification: CA = 5
- Total CA: 5 (same!)

**Verdict**: ✅ No coupling increase — safe to consolidate
```

Warning sign: `After: CA = 15 (was 5)` → ⚠️ high coupling increase; reconsider, or use a shared library instead of a shared service. A consolidated component that everything depends on is a bottleneck.

### Phase 5 — Consolidation approach

| Approach | Use when | Example |
| --- | --- | --- |
| **Shared service** | Logic changes frequently · complex operations · needs independent scaling · multiple deployment units will use it | A notification service called by many components |
| **Shared library** | Stable functionality · simple utilities · a compile-time dependency is acceptable · no independent deployment needed | Validation utilities as a package |
| **Component merge** | Highly related functionality · low coupling impact · the same deployment unit is fine | Merge 3 notification components into 1 |

**Decision tree:**

```
Found common pattern?
├─ YES → Analyze functionality
│   ├─ Similar enough?
│   │   ├─ YES → Assess coupling
│   │   │   ├─ CA increase acceptable? ──YES──▶ ✅ Consolidate
│   │   │   └──────────────────────────NO───▶ ⚠️ Reconsider, or use a shared library
│   │   └─ NO → ❌ Don't consolidate
└─ NO → No consolidation needed
```

### Output

```markdown
## Consolidation Opportunities

| Common Functionality | Components   | Current CA | After CA | Feasibility | Recommendation                |
| -------------------- | ------------ | ---------- | -------- | ----------- | ----------------------------- |
| Notification         | 3 components | 5          | 5        | ✅ High     | Consolidate to shared service |
| Audit                | 3 components | 8          | 12       | ⚠️ Medium   | Consolidate, monitor coupling |
| Validation           | 2 components | 3          | 3        | ✅ High     | Consolidate to shared library |

## Consolidation Plan — Priority: High

**Notification Components** → `services/notification` (shared service)

Steps:
1. Create the new `services/notification` component
2. Move the common functionality out of the 3 components
3. Create an abstraction for content/templates
4. Update dependents to use the new service
5. Remove the old notification components

Expected Impact: ~4,500 statements consolidated · 3 components → 1 · CA unchanged at 5 · single place to maintain
```

### Fitness functions

```javascript
function checkCommonPatterns(components, exclusionList = []) {
  const leaves = {}
  for (const c of components) {
    const leaf = extractLeafNode(c.namespace)
    if (!exclusionList.includes(leaf)) (leaves[leaf] ??= []).push(c.name)
  }
  return Object.entries(leaves)
    .filter(([, comps]) => comps.length > 1)
    .map(([pattern, components]) => ({ pattern, components, suggestion: 'Consider consolidating these components' }))
}

function checkSharedClasses(components, exclusionList = []) {
  const usage = {}
  for (const c of components) for (const imp of c.imports) {
    if (!exclusionList.includes(imp)) (usage[imp] ??= []).push(c.name)
  }
  return Object.entries(usage)
    .filter(([, users]) => users.length > 1)
    .map(([cls, usedBy]) => ({ class: cls, usedBy, suggestion: 'Consider extracting to a shared component or library' }))
}
```

### Do's / Don'ts

**Do ✅** distinguish domain from infrastructure · analyze coupling impact **before** consolidating · consider both shared-service and shared-library approaches · look for namespace patterns **and** shared classes · verify the functionality is truly similar · compute CA before and after.

**Don't ❌** consolidate infrastructure functionality here · consolidate without a coupling analysis · assume every common pattern should be consolidated · ignore real functional differences · consolidate when the coupling increase is too high · mix domain and infrastructure in one analysis.

> Some duplication is acceptable if it keeps coupling low. That is a legitimate outcome of this pattern, not a failure.

---

## 3. Pattern 3 — Flattening / hierarchy

Ensure components exist **only as leaf nodes**, and remove orphaned classes from root namespaces.

### Concepts

| Type | Definition | Example | Should have code? |
| --- | --- | --- | --- |
| **Component** | Leaf node (deepest directory with source files) | `ss.survey.templates` | ✅ Yes |
| **Root namespace** | A namespace extended by a child node | `ss.survey` (has `.templates`) | ❌ No — files here are orphaned |
| **Subdomain** | Same thing as a root namespace | `ss.survey` | ❌ No |

**Orphaned class** = a source file in a root namespace. Problem: no definable component owns it. Solution: move it into a leaf node.

```
ss.survey/              ← Root namespace (extended by .templates)
├── Survey.js           ← Orphaned class
└── templates/          ← Component (leaf node)
    └── Template.js
```

### Phase 1 — Map the structure

Build the namespace tree, mark parent-child relationships and leaf nodes, then locate every source file and map it to its namespace.

```
ss.survey/              ← Root namespace (extended)
├── Survey.js           ← Orphaned
├── SurveyProcessor.js  ← Orphaned
└── templates/          ← Component
    ├── EmailTemplate.js
    └── SMSTemplate.js

ss.ticket/              ← Root namespace (extended)
├── Ticket.js           ← Orphaned
├── assign/             ← Component
└── route/              ← Component
```

### Phase 2 — Identify and classify orphaned classes

Classify each orphaned file as **shared code** (common utilities, interfaces, abstract classes), **domain code** (business logic that belongs in a component), or **mixed**. Assess impact: how many files, what functionality, which components depend on them.

```markdown
### Root Namespace: ss.survey
**Orphaned Files** (5):
- Survey.js (domain — survey creation)
- SurveyProcessor.js (domain — survey processing)
- SurveyValidator.js (shared — validation)
- SurveyFormatter.js (shared — formatting)
- SurveyConstants.js (shared — constants)

Classification: 2 domain files → belong in components · 3 shared files → belong in a .shared component
Dependencies: used by the ss.survey.templates component
```

### Phase 3 — Choose a flattening strategy

| Strategy | Action | Use when |
| --- | --- | --- |
| **1. Consolidate down** | Move leaf-node code up into the root namespace, making it the component | Leaf nodes are small and closely related |
| **2. Split up** | Move root-namespace code down into new leaf nodes | The root namespace holds distinct functional areas |
| **3. Extract shared** | Move shared code into a dedicated `.shared` component | The root namespace mixes shared utilities with domain code |

```
Found orphaned classes?
├─ YES → Related to the leaf components?
│         ├─ YES → Consolidate Down
│         └─ NO  → Distinct functional areas?
│                  ├─ YES → Split Up
│                  └─ NO  → Shared code? → Extract Shared
└─ NO → ✅ Structure is already flat
```

Compare the options explicitly, with effort and rationale, before choosing:

```markdown
### Root Namespace: ss.survey  (root: 5 orphaned files · leaf: ss.survey.templates, 7 files)

Option 1 — Consolidate Down ✅ Recommended
  Move templates code into ss.survey → single component. Effort: Low (7 files).
  Rationale: templates are small and related to survey functionality.

Option 2 — Split Up
  ss.survey.create (2) + ss.survey.process (1) + ss.survey.shared (3), keeping templates (7).
  Effort: High. Rationale: more granular, but likely over-engineering here.

Option 3 — Move Shared Code
  ss.survey.shared (3 files); domain stays in root (2); templates unchanged.
  Effort: Medium. Rationale: separates shared from domain but leaves the hierarchy.
```

### Phase 4–5 — Plan and execute

Plan: list the files to move, their target namespaces, the dependencies to update, and an effort/risk estimate. Execute: move the files, update imports and namespace declarations, adjust the directory structure, then run the tests and check for broken references.

### Canonical before/after patterns

**Simple consolidation**

```
Before                        After
ss.survey/                    ss.survey/            ← Component
├── Survey.js     ← Orphaned  ├── Survey.js
└── templates/    ← Component └── Template.js
    └── Template.js
```

**Functional split**

```
Before                         After
ss.ticket/    ← Root           ss.ticket/           ← Subdomain
├── Ticket.js ← Orphaned (45)  ├── maintenance/     ← Component
├── assign/   ← Component      ├── completion/      ← Component
└── route/    ← Component      ├── assign/          ← Component
                               └── route/           ← Component
```

**Shared code extraction**

```
Before                          After
ss.survey/    ← Root            ss.survey/          ← Component
├── Survey.js         (domain)  ├── Survey.js
├── SurveyValidator.js (shared) └── shared/         ← Component
└── templates/ ← Component          └── SurveyValidator.js
```

### Output

```markdown
## Component Hierarchy Issues

| Root Namespace | Orphaned Files | Leaf Components                 | Issue                | Recommendation   |
| -------------- | -------------- | ------------------------------- | -------------------- | ---------------- |
| ss.survey      | 5              | 1 (templates)                   | Has orphaned classes | Consolidate down |
| ss.ticket      | 45             | 2 (assign, route)               | Large orphaned code  | Split up         |
| ss.reporting   | 0              | 3 (tickets, experts, financial) | No issue             | ✅ OK            |

## Flattening Plan

### Priority: High
**ss.survey** → Consolidate Down — move 7 files from templates to root. Effort: 2–3 days. Risk: Low.

### Priority: Medium
**ss.ticket** → Split Up — create ss.ticket.maintenance (30 files), ss.ticket.completion (10), ss.ticket.shared (5).
Effort: 1 week. Risk: Medium.
```

### Fitness functions

```javascript
// No source code in root namespaces
function checkRootNamespaceCode(namespaces, sourceFiles) {
  const violations = []
  for (const ns of namespaces) {
    const hasChildren = namespaces.some(n => n.startsWith(ns + '.') || n.startsWith(ns + '/'))
    if (!hasChildren) continue
    const files = sourceFiles.filter(f => f.namespace === ns)
    if (files.length > 0) {
      violations.push({ namespace: ns, files: files.map(f => f.name), issue: 'Root namespace contains source files (orphaned classes)' })
    }
  }
  return violations
}

// Components only as leaf nodes
function validateComponentStructure(namespaces, sourceFiles) {
  const leafNodes = namespaces.filter(ns => !namespaces.some(n => n.startsWith(ns + '.') || n.startsWith(ns + '/')))
  return sourceFiles
    .filter(f => !leafNodes.includes(f.namespace))
    .map(f => ({ file: f.name, namespace: f.namespace, issue: 'Source file not in a leaf node (component)' }))
}
```

### Do's / Don'ts

**Do ✅** ensure components exist only as leaf nodes · remove orphaned classes from root namespaces · choose the strategy based on functionality · consolidate when functionality is related · split when it is distinct · extract shared code to `.shared` components · update all references · verify with tests.

**Don't ❌** leave orphaned classes in root namespaces · create components on top of other components · skip updating imports after moving files · flatten without analyzing impact · mix strategies inconsistently · ignore shared code · skip testing after refactoring.

---

## 4. Pattern 5 — Domain identification and grouping

Group components into logical **domains** (business areas) to prepare for domain-aligned units.

> Run `coupling-analysis.md` (Pattern 4) first — the dependency graph is a primary input here. And ground the grouping in business language with `domain-boundaries.md`.

### Concepts

A **domain** represents a distinct business capability, contains related components, has clear boundaries and responsibilities, and could become a separately deployed unit. The relationship is **one-to-many**: one domain contains multiple components.

Domains are physically manifested through the **namespace structure**:

```
Before domain alignment          After domain alignment
services/billing/payment         services/customer/billing/payment
services/billing/history         services/customer/billing/history
services/customer/profile        services/customer/profile
services/supportcontract         services/customer/supportcontract
```

### Phase 1 — Identify business domains

Four complementary strategies:

1. **Business capability analysis** — what capabilities does the system provide? Each capability is a candidate domain.
2. **Vocabulary analysis** — group components sharing business vocabulary ("billing", "payment", "invoice" → Financial).
3. **Relationship analysis** — components frequently used together, or sharing data and workflows, likely belong together.
4. **Stakeholder collaboration** — validate with product owners and business analysts. Their understanding defines the boundaries.

```markdown
## Identified Domains
1. **Ticketing** (ss.ticket) — creation, assignment, routing, completion, surveys, knowledge base
2. **Customer** (ss.customer) — profile, billing and payment, support contracts
3. **Reporting** (ss.reporting) — ticket, expert, and financial reports
4. **Admin** (ss.admin) — user maintenance, expert profile management
5. **Shared** (ss.shared) — login, notification
```

### Phase 2 — Group components into domains

For each component: what business capability does it support? What vocabulary does it use? What does it relate to? Assign it to the best-fitting domain.

**Edge cases:** *unclear assignment* → analyze relationships more deeply. *Fits multiple domains* → choose a primary, document the secondary. *Genuinely shared* → may belong to a Shared domain.

### Phase 3 — Validate the groupings

Check **cohesion** (shared vocabulary, used together, direct relationships), verify **boundaries** are clear and each component belongs to exactly one domain, assess **completeness** (every component assigned), and get **stakeholder validation**.

**Domain size guidelines:** small 2–4 components (may need consolidation) · medium 5–8 (ideal) · large 9–15 (monitor) · >15 (consider splitting). Overall: **3–7 domains** is ideal; >10 suggests merging, <3 suggests splitting.

### Phase 4 — Refactor namespaces for domain alignment

```markdown
| Component        | Current Namespace   | Target Namespace            | Action        |
| ---------------- | ------------------- | --------------------------- | ------------- |
| Billing Payment  | ss.billing.payment  | ss.customer.billing.payment | Add .customer |
| Billing History  | ss.billing.history  | ss.customer.billing.history | Add .customer |
| Customer Profile | ss.customer.profile | ss.customer.profile         | No change     |
| Support Contract | ss.supportcontract  | ss.customer.supportcontract | Add .customer |
| KB Maintenance   | ss.kb.maintenance   | ss.ticket.kb.maintenance    | Add .ticket   |
| Survey           | ss.survey           | ss.ticket.survey            | Add .ticket   |
```

Steps: update namespace declarations → update import statements in dependents → update the directory structure → run tests → update documentation.

### Phase 5 — Create the domain map

```
Customer Domain (ss.customer)      Ticketing Domain (ss.ticket)     Reporting Domain (ss.reporting)
├── Customer Profile               ├── Ticket Shared                ├── Reporting Shared
├── Billing Payment                ├── Ticket Maintenance           ├── Ticket Reports
├── Billing History                ├── Ticket Completion            ├── Expert Reports
└── Support Contract               ├── Ticket Assign                └── Financial Reports
                                   ├── Ticket Route
Admin Domain (ss.admin)            ├── KB Maintenance               Shared Domain (ss.shared)
├── User Maintenance               ├── KB Search                    ├── Login
└── Expert Profile                 └── Survey                       └── Notification
```

**Domain relationships:**

```
Ticketing  ─uses─▶ Shared (Login, Notification)
           ─uses─▶ Customer (Customer Profile)
Customer   ─uses─▶ Shared (Login, Notification)
Reporting  ─uses─▶ Ticketing (ticket data) · Customer (customer data) · Shared (Login)
```

### Output

```markdown
## Domain: Customer (ss.customer)

**Business Capability**: manages customer relationships, billing, and support contracts

**Components**: Customer Profile · Billing Payment · Billing History · Support Contract
**Component Count**: 4
**Total Size**: ~15,000 statements (18% of codebase)

**Domain Cohesion**: ✅ High
- Components share customer-related vocabulary
- Components are frequently used together
- Direct relationships between components

**Boundaries**:
- Clear separation from Ticketing and Reporting
- Shared components (Notification) used by all domains
```

### Fitness functions

```javascript
// Components belong to the domain they were assigned
function validateDomainNamespaces(components, domainRules) {
  return components
    .map(c => ({ c, domain: identifyDomain(c.namespace), expected: domainRules[c.name] }))
    .filter(({ domain, expected }) => domain !== expected)
    .map(({ c, domain, expected }) => ({ component: c.name, currentDomain: domain, expectedDomain: expected, namespace: c.namespace }))
}

// No direct cross-domain dependencies (except into shared)
function enforceDomainBoundaries(components) {
  const violations = []
  for (const c of components) {
    const componentDomain = identifyDomain(c.namespace)
    for (const imp of c.imports) {
      const importedDomain = identifyDomain(imp)
      if (importedDomain !== componentDomain && importedDomain !== 'shared') {
        violations.push({ component: c.name, domain: componentDomain, importsFrom: imp, importedDomain, issue: 'Cross-domain direct dependency' })
      }
    }
  }
  return violations
}
```

### Typical domains in business applications

**Customer** (management, profiles, relationships) · **Product** (catalog, inventory, pricing) · **Order** (processing, fulfillment, shipping) · **Billing** (invoicing, payments, transactions) · **Reporting** (reports, analytics, dashboards) · **Admin** (user management, configuration) · **Shared** (login, notification, common utilities).

### Do's / Don'ts

**Do ✅** collaborate with business stakeholders · group by business capability, not technical layers · ensure domains represent distinct business areas · validate boundaries with stakeholders · refactor namespaces to align with domains · document domains clearly · use business language in domain names.

**Don't ❌** create domains from technical layers (services, controllers, models) · force components into domains where they do not fit · skip stakeholder validation · create too many small domains (aim for 3–7) · create monolithic domains · ignore components that do not fit (analyze *why*) · skip namespace refactoring — it is what makes the domains real.
