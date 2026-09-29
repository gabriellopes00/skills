# State Isolation Reference

P8 is the **critical** principle and the hardest to hold. Ambiguous ownership of data or names is the most common source of "works until it doesn't" integration bugs. This applies to relational tables, document collections, object-storage buckets, key-value namespaces, stream topics, and files.

## Table of Contents

1. The ownership rule
2. Naming conventions
3. Schema strategies (one database, many modules)
4. Cross-context references
5. Transaction ownership
6. Sub-unit ownership inside a context
7. Anti-patterns
8. Detection commands
9. Pre-merge gate

---

## 1. The ownership rule

**Each module owns its state and is the sole writer of it.**

- One shared database is **acceptable**. Sharing **tables** is not.
- Never put a foreign key across a module boundary.
- Never read another module's repository, store, bucket, or collection directly.
- Reference other contexts **by id**, validated in code rather than by the database.
- Every persisted fact has exactly **one** authoritative owner. If you cannot name the owner, that is the finding.

**Reads across a boundary** go through the owner's contract: a facade call, a published event, or a read model the owner maintains for consumers. If a consumer needs a joined view often, the right answer is usually a **projection the owner publishes**, not a cross-boundary join.

---

## 2. Naming conventions

**Every persisted concept is prefixed with its module name.** Generic names collide across modules and make ownership unknowable.

| Module | Entity name | Store name |
| --- | --- | --- |
| Identity | `IdentityUser` | `identity_users` |
| Identity | `IdentityProfile` | `identity_profiles` |
| Billing | `BillingPlan` | `billing_plans` |
| Billing | `BillingSubscription` | `billing_subscriptions` |
| Orders | `OrderRecord`, `OrderItem` | `order_records`, `order_items` |
| Content | `ContentArticle` | `content_articles` |

| ❌ Name | Problem |
| --- | --- |
| `User` | Which module? Identity? Billing? |
| `Plan` | Billing plan? Subscription plan? Project plan? |
| `Item` | Order item? Cart item? Inventory item? |
| `Profile` | User profile? Company profile? |

The same rule applies to queue names, topic names, bucket prefixes, cache-key namespaces, and metric names: `billing.subscription.activated`, `billing:plan:{id}`, `s3://app/billing/invoices/`.

---

## 3. Schema strategies (one database, many modules)

Pick one and apply it consistently.

### Option A — Single schema, module prefixes (simplest, recommended to start)

All models live in one schema, clearly prefixed and grouped by module with section comments.

```
── Identity module ───────────────
identity_users        (id, email, name, created_at, updated_at)
identity_profiles     (id, user_id, display_name)

── Billing module ────────────────
billing_plans         (id, name, price_in_cents, interval)
billing_subscriptions (id, user_id ← ID ONLY, plan_id → billing_plans.id, status)

── Orders module ─────────────────
order_records         (id, user_id ← ID ONLY, status, total)
order_items           (id, order_id → order_records.id, product_id, quantity, price)
```

Note the asymmetry: `billing_subscriptions.plan_id` **is** a foreign key (both tables belong to Billing), while `billing_subscriptions.user_id` is **just an id** (the user belongs to Identity).

### Option B — Schema per module

Each module gets its own database schema (`identity.users`, `billing.plans`). The database enforces part of the boundary for you, migrations stay per-module, and promoting a module to its own database later is close to a rename. Costs a little more setup and cross-schema querying discipline.

### Option C — Database per module

The end state of the evolution path — real physical isolation, independent scaling and backup, and no possibility of accidental joins. Only worth it when a module's own metrics justify it. Do not start here.

**For non-relational stores** the same three options map onto: collection prefixes / separate databases per module (document stores); key namespaces (KV and cache); bucket prefixes or separate buckets (object storage); topic prefixes (streams).

---

## 4. Cross-context references

```
✅ billing_subscriptions.user_id  → a plain id column, no FK, validated in code
❌ billing_subscriptions.user_id  → FOREIGN KEY REFERENCES identity_users(id)
```

**Why no cross-module foreign key:**

- It makes the two modules undeployable and unsplittable independently.
- It gives the database a rule that belongs to the domain.
- It silently authorizes reach-through joins, which then get written.
- It makes the referencing module's data lifecycle hostage to the other module's deletes.

**How to validate instead:** the referencing module checks existence through the owner's facade (or trusts an id it received in a command/event it already validated). Deletion in the owner publishes an event; the referencing module reacts with its own policy (cascade, soft-orphan, block).

**Value objects at the boundary.** Prefer a typed id over a raw string: `class Order { customerId: CustomerId }`. It documents which context the id belongs to and prevents mixing ids from different contexts.

---

## 5. Transaction ownership

- **One transaction = one aggregate.** A write that must be atomic stays inside one module and one aggregate.
- **Never open a transaction that spans module boundaries.** If two modules must both change, you need a **saga** (a sequence of local transactions with compensations) or the **outbox** (commit locally, publish, let the other side converge).
- Every cross-boundary write flow must have a documented answer to: *what happens if step 2 fails after step 1 committed?* If there is no answer, the design is incomplete.
- The **application layer owns the transaction**, not the repository and not the transport adapter.

---

## 6. Sub-unit ownership inside a context

Inside one bounded context with named sub-units, the same rule applies at a smaller scale:

- Each sub-unit **owns** its slice of model and persistence.
- Avoid one mega registration layer that wires **every** store and repository for **every** sub-unit in one place — that is a reach-through invitation.
- `shared/persistence` (or its equivalent) holds only the **connection/datasource**. Zero domain repositories.
- Cross-sub-unit reads go through the sibling's **internal facade**, never its repositories.

---

## 7. Anti-patterns

- **Reach-through persistence** — module A reads or writes module B's tables, buckets, or documents outside an agreed contract. The single most damaging violation.
- **Centralized data ownership** — one persistence module that registers and exposes *all* stores for *all* modules.
- **Shared entity classes** — two modules importing the same persistence entity. Even read-only, this couples their schemas forever.
- **Ambiguous global names** — `users`, `items`, `events` with no owner.
- **Cross-boundary foreign keys.**
- **Shared mutable process state** — an exported singleton cache or registry mutated by more than one module.
- **Accidental shared kernel** — entities that ended up shared with no documented owner and no decision behind it. (An **intentional** shared kernel of behavior-less state holders with documented ownership is legitimate — P19.)
- **Unscoped transactions** — writes spanning boundaries with no saga, outbox, or single-owner rule.

---

## 8. Detection commands

Adapt paths and file globs to the project. These are **signals**, not proof — confirm each hit by reading the code.

**Duplicate model / entity names across modules**

```bash
# Class-style domain entities
grep -rhoE "^export class [A-Z][A-Za-z]*" src/ | sort | uniq -d

# ORM/schema model declarations (Prisma-style)
grep -rh "^model " prisma/ | awk '{print $2}' | sort | uniq -d

# Duplicate physical table mappings
grep -rh '@@map(' prisma/ | grep -o '"[^"]*"' | sort | uniq -d
```

**Entities missing a module prefix** (single generic word, excluding technical suffixes)

```bash
grep -rnE "^export class [A-Z][a-z]+ " src/*/domain/ \
  | grep -vE "Error|Exception|Event|Command|Query|Handler|Dto|Module|Guard|Filter|Adapter|Port"
```

**Cross-module foreign keys** — list every relation and verify each stays inside one module

```bash
grep -rn '@relation\|REFERENCES\|FOREIGN KEY' prisma/ migrations/ \
  | sed 's/^/CHECK — must be within ONE module: /'
```

**Direct cross-module imports (bypassing the barrel/facade)**

```bash
grep -rn "from '@app/" src/ \
  | grep -v "/index" | grep -v "shared" | grep -v node_modules | grep -v ".spec."
```

**Shared mutable state (exported singletons)**

```bash
grep -rn "^export .*=.*new " src/ | grep -v test | grep -v node_modules
```

**Synchronous cross-module service calls (should be events or facade calls)**

```bash
grep -rn "await .*Service\." src/ | grep -v "this\." | grep -v test
```

**Cross-boundary store access from the wrong module**

```bash
# Any module referencing another module's table prefix
for m in identity billing orders content; do
  echo "── files outside $m/ that mention ${m}_ tables:"
  grep -rn "${m}_" src/ --include=*.ts | grep -v "src/$m/"
done
```

---

## 9. Pre-merge gate

Wire the checks into CI or a pre-commit hook. Treat duplicate names and cross-module foreign keys as **blocking**; treat import and sync-call findings as blocking once the team has adopted the convention.

```bash
#!/usr/bin/env bash
echo "🔍 State isolation checks..."
ERRORS=0

DUPES=$(grep -rh "^model " prisma/ 2>/dev/null | awk '{print $2}' | sort | uniq -d)
if [ -n "$DUPES" ]; then
  echo "❌ Duplicate model names: $DUPES"
  echo "   Fix: prefix each model with its module name (e.g. BillingPlan)"
  ERRORS=$((ERRORS + 1))
fi

MAP_DUPES=$(grep -rh '@@map(' prisma/ 2>/dev/null | grep -o '"[^"]*"' | sort | uniq -d)
if [ -n "$MAP_DUPES" ]; then
  echo "❌ Duplicate table mappings: $MAP_DUPES"
  ERRORS=$((ERRORS + 1))
fi

CROSS=$(grep -rn "from '@app/" src/ 2>/dev/null \
  | grep -v "/index" | grep -v "shared" | grep -v node_modules | grep -v ".spec.")
if [ -n "$CROSS" ]; then
  echo "⚠️  Deep cross-module imports (must go through the module barrel/facade):"
  echo "$CROSS"
  ERRORS=$((ERRORS + 1))
fi

if [ $ERRORS -gt 0 ]; then
  echo "❌ State isolation check failed with $ERRORS error(s)."
  exit 1
fi
echo "✅ State isolation checks passed."
```

More fitness functions — structural checks, boundary checks, and the audit severity tiers — are in `validation-and-testing.md`.
