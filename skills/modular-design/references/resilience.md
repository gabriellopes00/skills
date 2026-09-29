# Resilience Reference

Defend every external and inter-service call in layers, so a transient failure never takes down the flow or overwhelms a recovering dependency. This is P10 (Fail Independence) made concrete.

## Table of Contents

1. The layered model
2. Retries: capped exponential backoff + full jitter
3. Idempotency (a prerequisite for retries)
4. Circuit breaker
5. Retry budget, bulkhead, timeouts
6. Fallbacks
7. Durable execution (when)
8. Per-runtime notes
9. Policy checklist per adapter

---

## 1. The layered model

**Compose, do not substitute:**

```
timeout ⊃ circuit breaker ⊃ retry ⊃ call
```

- The **retry** handles a single transient error.
- The **circuit breaker** handles sustained failure — it stops the retry loop from hammering a dependency that is already down.
- The **timeout** bounds latency so a slow dependency cannot exhaust your capacity.
- The **bulkhead** isolates resources so one failing dependency cannot starve the others.

Use a proven library rather than hand-rolling this (Cockatiel or opossum in TypeScript, Polly in .NET, resilience4j in Java, tenacity + pybreaker in Python, failsafe-go / sony's gobreaker in Go). Hand-rolled retry loops are where the jitter and the budget get forgotten.

Apply the whole stack at **every adapter** — the ACL adapter is the natural home for the resilience policy, because it is already the one place that knows about the external system.

---

## 2. Retries: capped exponential backoff + full jitter

Naive retries cause the outages they try to fix: synchronized clients all retry at the same instant and produce a thundering herd against a recovering service. **Always use full jitter:**

```
sleep = random(0, min(cap, base * 2^attempt))
```

**Defaults to tune from:** `base ≈ 100ms` · `factor = 2` · `cap ≈ 30s` · `maxAttempts = 3–5`.

**Rules:**

- Honor `Retry-After` when the server sends it — and still add jitter on top.
- **Retry at one layer only.** Stacked retries multiply: 3 retries at three layers is 27 calls.
- Never retry on a 4xx that indicates a client error (400, 401, 403, 404, 422). Retry on 429, 5xx, connection errors, and timeouts.
- Log the attempt number and the reason on every retry, so retry storms are visible in telemetry.

```ts
// shape, not a library API
async function withRetry<T>(op: () => Promise<T>, o = { base: 100, cap: 30_000, max: 4 }) {
  for (let attempt = 0; ; attempt++) {
    try { return await op() }
    catch (err) {
      if (attempt >= o.max || !isTransient(err)) throw err
      const ceiling = Math.min(o.cap, o.base * 2 ** attempt)
      await sleep(Math.random() * ceiling)          // FULL jitter — random(0, ceiling)
    }
  }
}
```

---

## 3. Idempotency (a prerequisite for retries)

**Only retry operations that are safe to repeat.**

- Reads and naturally idempotent verbs (GET, HEAD, PUT, DELETE) are safe by construction.
- **Non-idempotent writes** (POST, charge, send, enqueue) require a client-supplied **idempotency key** that the server uses to dedupe. Use a stable business id: `idempotency_key = payment.id`, not a random per-attempt value.
- **Never enable retries on a write without server-side dedupe.** A request that timed out may already have executed — the timeout tells you nothing about whether it succeeded.
- If the external API offers no idempotency key, do not retry it. Instead: record the attempt, surface it for reconciliation, or wrap it in a workflow that can query-then-act.

The same rule governs event consumers — see `contracts-and-communication.md` §5.

---

## 4. Circuit breaker

Configure on a **sliding time-window error rate**, not an absolute count.

- Example policy: trip at **50% errors** over **≥20 requests** in a **10s window**.
- Also trip on a high **slow-call rate** — a dependency returning `200 OK` in 6 seconds is failing your SLA just as surely as one returning 500.
- **Open** → fail fast; do not even attempt the call, and do not let the retry loop run.
- **Half-open** → allow a small number of probe requests; close on success, re-open on failure.
- One breaker **per dependency** (and per operation if their failure modes differ), never one global breaker.
- Emit a metric on every state transition. A breaker that trips silently is an outage you will discover from users.

---

## 5. Retry budget, bulkhead, timeouts

**Retry budget.** Cap retries at roughly **10% of request volume** over a rolling window. Retries are load multipliers; without a budget, a partial outage becomes a full one.

**Bulkhead.** Isolate resource pools per dependency — separate connection pools, separate concurrency limits, separate queues — so one failing dependency cannot exhaust the threads or connections the others need.

**Timeout.** Set the request timeout to **2–5× the downstream p99**. Never wait forever. Every call gets one: HTTP requests, database queries, lock acquisitions, queue receives, and lambda invocations. An unbounded wait at a boundary is a design bug (P10).

Also bound the **total** time of a retried operation, not only each attempt — otherwise 4 attempts at a 30s timeout each is a 2-minute hang.

---

## 6. Fallbacks

**Every circuit breaker needs a named fallback** — never a raw exception surfaced to the user.

Acceptable fallbacks, roughly in order of preference:

1. **Stale cache read** — serve last-known-good data, flagged as stale where it matters. For idempotent reads this beats a 500 almost always.
2. **Outbox write / queue for later** — accept the request, persist the intent, converge asynchronously.
3. **Degraded response** — defaults, a reduced feature set, or an explicit "temporarily unavailable for this section" while the rest of the page works.
4. **Explicit, actionable error** — when there is genuinely nothing to serve, say what failed and what the user can do.

The fallback is part of the design, not an afterthought. Write it down next to the port it protects.

---

## 7. Durable execution (when)

Add a durable-execution engine (Temporal, Restate, DBOS, Step Functions, Azure Durable Functions) **only** for long-running, multi-step processes that must **resume exactly where they left off** after a crash.

*Example:* ingest → OCR → human review → ERP upsert → notify → close. Per-step retries and idempotency come built in.

**Keep ordinary request/response logic out of it.** It is a heavy tool with real operational and cognitive cost.

- If the engine is owned by another team, treat it as an external system behind a `WorkflowOrchestrationPort` (ACL) — and keep your own integrations (ERP, storage, notifications) as capabilities the workflow *invokes*, so you retain control of vendor-agnosticism.
- For your own background work (outbox relay, jobs, crons), start simple with a **database-backed queue**. Reach for a heavier engine only when scale or step complexity justifies it.

---

## 8. Per-runtime notes

| Runtime | Notes |
| --- | --- |
| **HTTP server** | Full stack applies. Watch total request budget: client timeout must exceed your internal retry ceiling, or the client gives up while you are still retrying. |
| **Serverless** | The platform already retries invocations — count that as a retry layer and reduce your own, or you multiply. Function timeout caps the whole operation, so keep retry ceilings well inside it. Circuit-breaker state does not survive between cold containers: keep it in a shared store (Redis/Valkey) if it must be accurate. |
| **Queue worker** | The broker's redelivery *is* your retry mechanism — configure visibility timeout and max receives instead of an in-process retry loop, and use a **dead-letter queue** as the terminal fallback. Consumers must be idempotent. |
| **Scheduled job** | The next run is the retry. Make the job resumable and idempotent; record progress so a crashed run does not redo completed work. |
| **CLI / batch** | Bound total runtime; make it re-runnable safely; report partial progress on failure so the operator can resume rather than restart. |

---

## 9. Policy checklist per adapter

For every port/adapter, record and review:

- [ ] **Timeout** — value, and how it relates to the downstream p99.
- [ ] **Retry** — attempts, base, cap, jitter (full), and which errors are retryable.
- [ ] **Idempotency** — is the operation idempotent? If it is a write, what is the idempotency key?
- [ ] **Circuit breaker** — window, error-rate threshold, slow-call threshold, half-open probe count.
- [ ] **Retry budget** — the cap as a fraction of traffic.
- [ ] **Bulkhead** — the pool/concurrency limit for this dependency.
- [ ] **Fallback** — the named degraded behavior, written down.
- [ ] **Telemetry** — metrics for attempts, breaker state transitions, fallback invocations, and timeouts.
- [ ] **Total operation budget** — the ceiling on the whole retried operation, not just each attempt.

An adapter with no answer for any of these rows is not ready for production.
