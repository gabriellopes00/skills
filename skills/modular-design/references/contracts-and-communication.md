# Contracts & Communication Reference

How modules talk to the outside world and to each other without coupling. Applies identically to HTTP services, workers, serverless functions, and CLIs — only the transport adapter changes.

## Table of Contents

1. Ports & Adapters (the Anti-Corruption Layer)
2. Choosing sync vs async
3. Inter-module communication: events + the transactional outbox
4. Event contracts and naming
5. Idempotent consumers
6. Choosing an event transport
7. Real-time server-to-client push
8. Contract versioning and evolution

---

## 1. Ports & Adapters (the Anti-Corruption Layer)

Every external system — third-party API, ERP, ticketing system, object storage, payment gateway, AI provider, identity provider, workflow engine, **and even an internal service owned by another team** — sits behind a **Port** (an interface in *your domain's* language) implemented by an **Adapter** (infrastructure).

- The **domain declares the port**: `ErpLedgerPort`, `TicketSourcePort`, `DocumentStoragePort`, `AiAssistPort`, `IdentityProviderPort`, `WorkflowOrchestrationPort`, `NotifierPort`.
- The **adapter implements it**: `OmieAdapter`, `S3StorageAdapter`, `ZendeskAdapter`. The adapter **translates** the external model into your domain model and back, so **no vendor type leaks inside**.
- Dependencies point inward: the adapter knows the port; the port never knows the adapter (Dependency Inversion).
- Swapping a vendor = writing one new adapter behind the same port. The core does not change.

This is what makes a platform vendor-agnostic as a **structural property**, not a promise.

```ts
// domain — expressed as a CAPABILITY, never as the provider's API shape
export interface ErpLedgerPort {
  postLedgerEntry(entry: LedgerEntry): Promise<LedgerReceipt>
  fetchNextDemand(): Promise<Demand | null>
}

// infrastructure — the ONLY place that knows the vendor exists
export class OmieAdapter implements ErpLedgerPort {
  async postLedgerEntry(entry: LedgerEntry): Promise<LedgerReceipt> {
    const payload = toOmiePayload(entry)          // translate OUT
    const res = await this.http.post('/lancamento', payload)
    return toLedgerReceipt(res)                    // translate IN
  }
}
```

**Rules:**

- Keep ports expressed as capabilities ("post a ledger entry", "fetch the next demand", "store this document"), never as `callOmieEndpointX`.
- The port's types are **your** types. If a vendor enum appears in a port signature, the ACL has failed.
- Provide an in-memory / fake adapter for every port — that is what makes domain tests fast and hermetic.
- The composition root is the only place that binds a port to a concrete adapter.
- If the external system is owned by another team inside your company, it still gets a port. "Internal" is not the same as "stable."

---

## 2. Choosing sync vs async

| Situation | Choice | Why |
| --- | --- | --- |
| Inside one aggregate | **Synchronous**, one atomic transaction | Invariants must hold immediately |
| Across modules, needs an immediate answer to proceed | Synchronous call **through the facade contract** | Accept the coupling; document the timeout and fallback |
| Across modules, tolerates eventual consistency | **Event via the outbox** | Audit, gamification, notifications, projections, queue reordering |
| Multiple consumers need to react | **Event** | One publisher, N independent consumers |
| Long-running multi-step process that must resume after a crash | **Durable execution** (see `resilience.md`) | Per-step retries and idempotency built in |

Not everything is an event. Keep the things that need atomicity — approving a payment and changing the request status — inside the aggregate. Use events for what tolerates eventual consistency.

Every synchronous cross-module or cross-service call needs an explicit **timeout**, **retry policy**, and **failure behavior** declared with it.

---

## 3. Inter-module communication: events + the transactional outbox

**Never call directly into another module's internal service.** Publish an event.

Publish reliably with the **transactional outbox**: write the business change and the event row in the **same transaction**; a relay reads unpublished rows afterwards and pushes them to the bus. This removes the dual-write problem — the classic failure where the data commits but the notification is lost (or vice versa).

```
┌─────────────── ONE ATOMIC TRANSACTION ───────────────┐
│  UPDATE payments SET status='APPROVED' WHERE id=…    │
│  INSERT INTO outbox (id, type, payload, published_at)│
└──────────────────────────────────────────────────────┘
                      │
                 relay polls unpublished rows
                      ▼
                 [ message bus ]
                      ▼
        idempotent consumer(s) in other modules
```

**Outbox row shape:** `id` (the event id, used for dedupe) · `aggregate_id` · `event_type` · `payload` (JSON) · `occurred_at` · `published_at` (null until published) · `attempts`.

**Where the relay runs, per runtime shape:**

| Runtime | Relay |
| --- | --- |
| HTTP server (one deploy) | Background worker inside the same process — no extra infrastructure |
| Scaled-out servers | A dedicated relay process/worker; the contract does not change |
| Serverless | A scheduled function polling the outbox table, or a CDC stream on the table |
| Queue worker | The worker's own loop drains the outbox after each unit of work |
| CLI / batch | The job publishes the outbox at the end of its run |

The relay is an implementation detail. Moving it from "in-process worker" to "dedicated worker" is a deployment change, not a contract change — which is exactly the point of the pattern.

---

## 4. Event contracts and naming

- Name events **`module.aggregate.action`** in past tense: `review.payments.approved`, `identity.user.created`, `billing.subscription.activated`, `erp.upsert.succeeded`.
- Payloads carry **serializable primitives and ids only** — never entity references, never rich domain objects.
- Events are **immutable** once created.
- **Version** events when their schema changes.
- Define event contracts in a **shared contracts area** that both publisher and consumer depend on — this is a legitimate, intentional shared kernel (P19), limited to contract types.

```ts
// shared contract — the published language between modules
export interface DomainEvent {
  readonly eventId: string        // dedupe key
  readonly aggregateId: string
  readonly eventType: string      // 'billing.subscription.activated'
  readonly occurredAt: Date
  readonly version: number
  readonly payload: Record<string, unknown>
}

export interface EventPublisher {
  publish<T extends Record<string, unknown>>(eventName: string, payload: T): Promise<void>
}
```

```ts
// a concrete event contract
export class BillingSubscriptionActivated implements DomainEvent {
  readonly eventType = 'billing.subscription.activated'
  readonly version = 1
  constructor(
    readonly eventId: string,
    readonly aggregateId: string,
    readonly occurredAt: Date,
    readonly payload: { subscriptionId: string; planId: string; userId: string },
  ) {}
}
```

**Rules for event contracts:**

- Dot notation, `module.aggregate.action`.
- Only serializable primitive data in the payload.
- Never include domain entity references — use ids.
- Version on schema change; support the old version until consumers migrate.
- Enrich the payload with what consumers will need, so handlers do not have to call back for data that could have travelled with the event.

---

## 5. Idempotent consumers

The outbox guarantees **at-least-once** delivery. Therefore **every consumer must be idempotent** — a redelivery must produce the same end state, not a second side effect.

**Deduplication key:** the `eventId`, or `aggregateId + eventType` when the event is naturally once-per-aggregate.

```
on(event):
  if processed_events.contains(event.eventId): return        # already handled
  within transaction:
      apply the business change
      processed_events.insert(event.eventId, now())
  log(event.eventType, event.aggregateId, 'processed')
```

**Rules:**

- Check whether the action was already performed *before* executing it.
- Do the dedupe insert in the **same transaction** as the effect, or the consumer is not actually idempotent.
- Log every event processed, with type and aggregate id, for observability.
- Prefer naturally idempotent operations (upsert by id, set-to-value) over increments.
- Give the dedupe table a retention policy; it grows forever otherwise.

---

## 6. Choosing an event transport

| Criterion | Kafka | Managed queue (SQS/PubSub) | Redis/Valkey pub/sub | DB-backed queue | In-memory |
| --- | --- | --- | --- | --- | --- |
| **Delivery** | At-least-once | At-least-once | At-most-once | At-least-once | Best-effort |
| **Ordering** | Per-partition | FIFO queues | None | By id / timestamp | Synchronous |
| **Persistence** | Yes (configurable retention) | Yes (days) | No | Yes | No |
| **Throughput** | Very high | High | Very high | Moderate | n/a |
| **Ops complexity** | High | Low | Low | Minimal | Minimal |
| **Best for** | Event sourcing, very high scale | Cloud-native, simple flows | Fan-out to connected clients | Starting point; no new infrastructure | Dev/test only |

> ⚠️ **Never use in-memory events for production inter-module communication.** They do not survive a restart, do not cross instances, and have no delivery guarantee. In-memory is for local development and tests only.

**Recommendation:** start with a **database-backed queue** (the outbox table itself is already there) or the managed queue of your cloud. Move to Kafka only when you have genuinely outgrown the simpler option. Select the transport behind the same `EventPublisher` port so swapping it is a composition-root change:

```ts
// composition root — the only place a transport is named
const publisher: EventPublisher =
  config.EVENT_DRIVER === 'kafka' ? new KafkaEventPublisher(producer)
: config.EVENT_DRIVER === 'sqs'   ? new SqsEventPublisher(client, queueResolver)
: config.EVENT_DRIVER === 'redis' ? new RedisEventPublisher(redis)
:                                   new InMemoryEventPublisher(bus)   // dev/test only
```

Each adapter maps the generic event onto its transport (Kafka: topic per event family, key = event name, headers carry type and timestamp; queue: message attributes carry the type; pub/sub: channel = event name).

---

## 7. Real-time server-to-client push

For server-to-client push (live queues, notifications, dashboards, token streaming), prefer **SSE (Server-Sent Events)**: plain HTTP, automatic reconnection with `Last-Event-ID`, works through proxies, WAFs, and load balancers, needs no special infrastructure. In the browser it is `EventSource`.

- **Multi-instance:** fan out through a pub/sub channel (Redis/Valkey) so any instance can push to the clients connected to it. A single instance needs no pub/sub.
- **Reconnection** should use the same backoff + jitter policy as the rest of the system (`resilience.md`).
- **Client caching:** SSE signals "something changed"; let the client's cache layer refetch or patch. With a query cache, update via `setQueryData` / `invalidateQueries` on the event.
- Use **WebSocket** only when you genuinely need low-latency **bidirectional** traffic (collaborative editing, multiplayer, live cursors). It costs sticky sessions plus a pub/sub backbone. In roughly 80% of "we need WebSocket" cases, SSE is enough.
- **Serverless:** SSE fits poorly with short-lived function timeouts — use the platform's managed WebSocket API or a push service instead.
- **No client at all** (workers, batch): this section does not apply; emit metrics and events instead.

```
[browser A]──SSE──┐                      ┌──▶[instance 1]──┐
[browser B]──SSE──┼──▶ load balancer ──▶ ┤                 ├──▶ pub/sub channel ◀── outbox relay
[browser C]──SSE──┘                      └──▶[instance 2]──┘
```

---

## 8. Contract versioning and evolution

A contract crosses a boundary, so changing it is a **breaking change for someone**. Treat every published contract — HTTP API, event schema, facade signature, message payload — the same way.

**Additive changes are safe:** adding an optional field, adding a new event type, adding a new operation.

**Breaking changes need a version:** removing or renaming a field, changing a type, changing the meaning of a value, tightening validation.

**Evolution strategy:**

1. Publish the new version alongside the old (`v1` and `v2` events; `/v2` endpoint; a new facade method).
2. Migrate consumers one at a time.
3. Instrument the old version so you can *see* when it stops being used.
4. Remove the old version only after the telemetry shows zero traffic.

**Document per contract:** inputs, outputs, errors, consistency guarantee (immediate or eventual), delivery guarantee (at-least-once, at-most-once), timeout, retry behavior, and idempotency requirements. A contract without failure semantics is incomplete.
