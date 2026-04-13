# The Pit of Success: Engineering Defaults for Distributed Systems

> **What this is:** A shared vocabulary for reliability — a common language and a tiered set of patterns that reduce analysis paralysis during development.
>
> The bet: invest in reliable infrastructure and code-level practices now. It pays off in the long run: fewer race conditions, duplicates, and firefighting as the codebase scales.

---

## Entropy Grows

This is the natural state of all systems. Over time, complexity grows and changes compound in unpredictable ways.

A typical sync handler in a young codebase might touch 4–5 services with a handful of I/O calls. Three months later, the same handler — organically extended as features were added — can easily touch 6+ services with 26–62 I/O calls, loops per entity, and multiple code paths. The code is still "correct," but it now fails reliably in production because of the math below.

---

## Napkin Math for Network Reliability

These results are genuinely non-intuitive. A "hop" here means any RPC or database call.

**Formula:** `P(day ok) = p^(h×n)` · `E(fails/day) = n × (1 - p^h)`

### At 99.99% per hop (generous — AWS Load Balancer SLA level)

**Chance of a clean day:**

| ops/day | 5 hops | 25 hops | 50 hops |
|---|---|---|---|
| 100 | 95% | 78% | 61% |
| 500 | 78% | 29% | 8% |
| 1,000 | 61% | 8% | 0.7% |

**Expected failures per day:**

| ops/day | 5 hops | 25 hops | 50 hops |
|---|---|---|---|
| 100 | 0.05 | 0.25 | 0.50 |
| 500 | 0.25 | 1.25 | 2.49 |
| 1,000 | 0.50 | 2.50 | 4.99 |

### At 99.9% per hop (realistic — what AWS SQS itself promises)

**Chance of a clean day:**

| ops/day | 5 hops | 25 hops | 50 hops |
|---|---|---|---|
| 100 | 61% | 8% | 0.7% |
| 500 | 8% | ~0% | ~0% |
| 1,000 | 0.7% | ~0% | ~0% |

**Expected failures per day:**

| ops/day | 5 hops | 25 hops | 50 hops |
|---|---|---|---|
| 100 | 0.50 | 2.47 | 4.88 |
| 500 | 2.50 | 12.35 | 24.40 |
| 1,000 | 4.99 | 24.70 | 48.79 |

These are **infrastructural constraints** — a consequence of running a distributed system, not a bug in your code.

And then there are silent data corruption errors, which don't throw exceptions and are usually detected via user complaints.

### Effect of retries (at 99.9%, 50 hops)

| Attempts | Error % |
|---|---|
| 1 | 4.88% |
| 2 | 0.238% |
| 5 | 0.0000028% |
| 10 | ~0% |

**How to achieve higher availability from lower-availability parts: idempotency + retries.**

---

## The Strategy: Managing Entropy

Distributed systems guarantee partial failure. Messages will be duplicated, delivered out of order, or dropped entirely. Code complexity grows naturally.

### The High-Leverage Principles

**1. Naming is the highest-ROI activity**

"Ubiquitous Language" is not just for DDD purists — it's an error-prevention tool.

Move from generic CRUD (`HandleEntityToggleChanged`) to domain intent (`ProvisionEntity`, `DeprovisionEntity`). Logic bugs become obvious syntax errors. Non-technical stakeholders can spot flaws in the design phase, saving expensive refactors later.

**2. Source of truth over derived data**

A read model, a cache, a replicated column in another service's database — these are all projections that lag behind reality. When a service makes a decision based on derived data, it is gambling on freshness. Design so that critical decisions flow through the source of truth.

**3. Linearization as a service**

Concurrency is a hard problem. Trying to manage it with application-level locks and compare-and-swap is high-risk and high-effort. Consider infrastructure that provides single-writer semantics by design: FIFO queues, partitioned streams (Kafka), actor models, or routing messages by entity ID so all mutations for a given entity are serialized.

**4. Push observability**

Metrics must be loud. If a dashboard requires manual inspection to be useful, it is effectively dead. Move from "investigative logging" to "status signals." A feature-level dashboard should answer *"Is this broken?"* in under 5 seconds.

---

## The Convergence Spectrum

When a business operation touches multiple services, how do you ensure they eventually agree? Safety and complexity increase together as you move down this list.

| Level | Name | In a sentence |
|---|---|---|
| **L1** | Fire-and-Forget | Fast, cheap, non-critical. Best effort with no recovery. |
| **L2** | Best Effort | Apply safely if it runs — idempotent, atomic, no rollback needed. |
| **L3** | Scheduled Repair | Fix later on a timer (hourly/nightly reconciliation job). |
| **L4** | Continuous Reconciliation | Fix continuously: on event + L3 as backstop. |
| **L5** | Strong Consistency | Prevent divergence entirely via single-DB transaction or distributed consensus (Raft/Paxos). |

**Default recommendation: start at L2.** It provides the best balance of reliability and engineering effort for most features.

**Why L2 makes sense early:** Small data volumes mean low contention, error probability is manageable, and manual repair is still practical. You can focus on doing safe writes — idempotent, version-guarded, deterministic IDs — without building continuous reconciliation infrastructure.

**L3 is a hard problem.** You may need data from multiple sources without point-in-time semantics, with updates potentially in-flight during reconciliation. Writes to sync can fail — has to be idempotent. Good news: many usage patterns have natural quiet periods (e.g., nightly). "Good enough" reconciliation can be acceptable.

**L1 is valid when explicit.** Some data genuinely doesn't need recovery. Claiming L1 consciously is fine; accidentally operating at L1 while thinking you're at L2 is a bug.

---

## What Defines Best Effort (L2)

A feature operating at L2 should have:

- **Bounded retries + dead letter queue** — the handler gets multiple chances; failures that exhaust retries are quarantined, not silently dropped
- **Atomic updates** — concurrent writes don't corrupt each other
- **Idempotent writes** — retries produce the same result (via version locks or deterministic IDs)
- **Observability** — business-level metrics, correlation IDs, distributed tracing

---

## Pattern Index

### Tier 1 — Highest ROI (easy wins)

#### Atomic Updates

**Problem:** Read → modify → write creates lost update race conditions. Data is only consistent inside a transaction. The moment bytes hit the wire and the row is unlocked, treat it as potentially changed.

**Anti-pattern:**
```python
entity = get_entity(id)       # stale by definition
entity.field = new_value      # race condition
save_entity(entity)           # might overwrite a concurrent write
```

**Golden path:** Express mutations as deltas — `$set`, `$addToSet`, `increment` — not full-state replacements. Let the database apply the delta atomically.

```python
update_entity(id, set__field=new_value)  # atomic, no read needed
```

*Reference: Kleppmann, DDIA Ch. 7; any ACID database documentation*

---

#### Deterministic IDs

**Problem:** Message reprocessing creates duplicate entities when IDs are generated randomly on each execution.

**Solution:** Generate IDs deterministically — same input always produces the same ID. Concurrent or retried creation becomes idempotent.

```python
import hashlib

def deterministic_id(namespace: str, source_id: str) -> str:
    return hashlib.sha256(f"{namespace}:{source_id}".encode()).hexdigest()[:24]

# deterministic_id("team", "external-team-42") → always the same value
```

*Reference: UUID v5 (RFC 4122); SHA truncation*

---

#### Immutability by Default

**Problem:** Mutable shared state is a source of subtle, hard-to-reproduce bugs.

**Solution:** Prefer immutable value objects. Construct a new instance with changed fields rather than mutating in place.

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True)
class TeamContext:
    team_id: str
    org_id: str

updated = replace(original, org_id=new_org_id)
```

*Reference: Fowler, "Value Object"; Eric Evans, DDD*

---

### Tier 2 — Essential for Reliability (L2 Foundation)

#### Optimistic Concurrency

**Problem:** Concurrent modifications silently overwrite each other.

**Solution:** Read with a version number, write with a version check, retry on conflict. No locks held between read and write.

```python
# Read
entity = db.find_one(id)          # entity.version == 5

# Write — only succeeds if version hasn't changed
result = db.update_one(
    {"id": id, "version": 5},
    {"$set": {"field": new_value}, "$inc": {"version": 1}}
)
if result.matched_count == 0:
    raise ConflictError("concurrent modification detected — retry")
```

*Reference: Fowler, "Optimistic Offline Lock"; RFC 7232 (ETags)*

---

#### Entity Versioning

**Problem:** Consumers of your data can't tell if they're looking at a stale version.

**Solution:** Include `version` (monotonically incrementing integer) and `updated_at` (timestamp) on every new entity by default. Low upfront cost; huge savings when other teams start consuming your data or when you need to implement optimistic concurrency later.

```python
@dataclass
class Entity:
    id: str
    version: int = 0
    updated_at: datetime = field(default_factory=datetime.utcnow)
    # ... business fields
```

---

#### Anti-Corruption Layer

**Problem:** External DTOs and third-party formats leak into your domain, forcing defensive null checks everywhere.

**Solution:** Validate at the boundary. Inside the boundary, work with guaranteed-valid types.

```python
def handle_message(raw_payload: dict) -> None:
    dto = ExternalDto.parse(raw_payload)      # raises if invalid
    assert dto.entity_id, "entity_id required"
    # From here: dto is trusted, no more null checks
    process(dto.entity_id, dto.event_type)
```

*Reference: Fowler, "Anti-Corruption Layer"; Azure Architecture docs*

---

### Tier 3 — Infrastructure & Scalability

#### Funnel (Single-Writer per Entity)

**Problem:** Concurrent mutations to the same entity from multiple consumers cause conflicts even with optimistic locking.

**Solution:** Route all events for a given entity to the same consumer instance. Infrastructure handles serialization — you write code as if there's only one writer.

Approaches:
- **Kafka:** partition by entity ID — all events for entity X go to partition hash(X)
- **SQS FIFO:** `MessageGroupId = entity_id`
- **RabbitMQ:** consistent hash exchange routing on entity ID
- **Actor model:** one actor per entity (Akka, Orleans, Dapr)

*Reference: Kleppmann, DDIA Ch. 6; Hewitt, "Actor Model"*

---

#### Inbox (Deduplication on Consume)

**Problem:** At-least-once delivery means your consumer will occasionally receive the same message twice. Without deduplication, side effects execute multiple times.

**Solution:** Before doing any work, check if this message ID has already been processed. If yes, acknowledge and return — do nothing.

```python
def handle(message: Message) -> None:
    if db.exists("processed_messages", message.id):
        return  # already done, safe to ack

    # Do the work
    apply_event(message.payload)

    # Record as processed (ideally in the same transaction as the work)
    db.insert("processed_messages", {"id": message.id, "at": now()})
```

*Reference: Stripe, "Designing APIs with Idempotency"; Brandur, "Idempotency Keys in Postgres"; libraries: Brighter, MassTransit, NServiceBus*

---

#### Outbox (Reliable Publish on Write)

**Problem:** Writing to a database and publishing to a message queue are two separate operations. A crash between them creates silent inconsistency — the DB write succeeded but the event was never published (or vice versa).

```python
# BROKEN — not atomic
db.save(entity)          # succeeds
queue.publish(event)     # crash here → event lost forever
```

**Solution:** Write the outbox entry into the same database transaction as the business state. A separate relay process reads the outbox and publishes to the queue.

```sql
BEGIN;
  INSERT INTO entities (id, ...) VALUES (...);
  INSERT INTO outbox (id, event_type, payload, created_at)
    VALUES (gen_random_uuid(), 'entity.created', '{"id": ...}', now());
COMMIT;
-- Either both committed or neither. ACID guaranteed.
-- Relay process reads outbox and publishes to queue asynchronously.
```

*Reference: Richardson, "Transactional Outbox Pattern"; Kleppmann, DDIA Ch. 11; libraries: Brighter, Wolverine, MassTransit, NServiceBus*

---

#### Bounded Parallelism

**Problem:** Unbounded fan-out exhausts thread pools, connection pools, and downstream rate limits.

```python
# BROKEN — 1,000 concurrent connections
tasks = [process(item) for item in items]
await asyncio.gather(*tasks)
```

**Solution:** Cap concurrency explicitly.

```python
import asyncio

semaphore = asyncio.Semaphore(5)

async def bounded(item):
    async with semaphore:
        return await process(item)

await asyncio.gather(*[bounded(item) for item in items])
```

---

#### Background Jobs

**Problem:** Fire-and-forget (`Task.Run`, `asyncio.create_task` with no tracking) loses work silently on process restart or crash.

**Solution:** Use a durable job queue (Hangfire, Celery, BullMQ, Sidekiq) or a message queue. The job is persisted before the process handles it; the worker can restart and resume.

*Reference: Hohpe & Woolf, "Guaranteed Delivery"; hangfire.io*

---

### Tier 4 — Code Clarity & Error Handling

#### Prepare-Decide-Commit (Impure/Pure/Impure Sandwich)

**Problem:** Interleaved reads and writes make failures hard to reason about and business logic hard to test.

**Solution:** Structure every operation in three phases:

```
PHASE 1: PREPARE  — fetch all context, no writes
PHASE 2: DECIDE   — pure business logic, no I/O
PHASE 3: COMMIT   — atomic writes
```

```python
# 1. PREPARE — gather all context upfront
context = OperationContext(
    entity = db.get_entity(entity_id),
    mapping = db.get_mapping(org_id),
)

# 2. DECIDE — pure function, easily unit-tested without mocks
decision = decide_sync_action(context)
if not decision.should_proceed:
    return Result.skipped(decision.reason)

# 3. COMMIT — write once at the end
db.upsert_entity(decision.entity)
db.update_mapping(decision.mapping_delta)
```

**Why it works:** The pure `decide_sync_action` function can be tested with plain data. No mocks. No I/O in tests. All the complexity lives in a function with no side effects.

*Reference: Seemann, "Impureim Sandwich" (2020); Bernhardt, "Functional Core, Imperative Shell"*

---

#### Result Types

**Problem:** Using exceptions for expected business outcomes (`EntityNotFound`, `ValidationFailed`) hides control flow and makes error tracking harder.

**Solution:** Return explicit typed results.

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")

@dataclass
class Result(Generic[T]):
    value: T | None = None
    error: str | None = None
    skip_reason: str | None = None

    @staticmethod
    def ok(value: T) -> "Result[T]": return Result(value=value)

    @staticmethod
    def skipped(reason: str) -> "Result[T]": return Result(skip_reason=reason)

    @staticmethod
    def failed(error: str) -> "Result[T]": return Result(error=error)

    @property
    def is_ok(self): return self.value is not None
```

*Reference: Wlaschin, "Railway Oriented Programming"; Rust Book Ch. 9 — `Result<T, E>`*

---

#### Circuit Breaker

**Problem:** When a dependency is down, every request waits for a timeout before failing. Thread pools drain. Your service slows to a crawl because it's being polite to a corpse.

**Solution:** Track failure rate. Trip the circuit when it exceeds a threshold. While open, fail immediately without attempting the call. Periodically let one probe request through to check if the dependency has recovered.

```
States:
  CLOSED     → requests flow through normally
  OPEN       → fail immediately, no call made (after failure threshold exceeded)
  HALF-OPEN  → one probe request allowed through
               success → CLOSED
               failure → OPEN again
```

*Reference: Nygard, "Release It!" (2007); Fowler, "Circuit Breaker"*

---

#### Saga (Distributed Transactions)

**Problem:** You need atomicity across two or more services/databases that cannot share a transaction.

**Solution:** Break the operation into steps. Each step registers a compensating action that undoes it. On failure, execute compensations in reverse order.

```python
compensations = []

try:
    # Step 1
    org_id = create_org(data)
    compensations.append(lambda: delete_org(org_id))

    # Step 2
    mapping_id = create_mapping(org_id, data)
    compensations.append(lambda: delete_mapping(mapping_id))

    # Step 3
    send_event(org_id)
    # No compensation for event — use idempotency on consumer side

except Exception:
    for compensate in reversed(compensations):
        try:
            compensate()
        except Exception as e:
            log.error("compensation failed", exc=e)
    raise
```

**Caution:** Compensations can also fail. For L3/L4 systems, pair with the Outbox pattern and idempotent compensations.

*Reference: Garcia-Molina & Salem, "Sagas" (1987); McCaffrey, "Distributed Sagas" (2015)*

---

## Wrapping Up

No one-size-fits-all solution exists. Explicit trade-offs are better than implicit "high consistency." It enables realistic expectations.

Some features genuinely belong at L1. Others need L4. **The goal isn't uniform consistency — it's conscious choice over accidental complexity.**

| Takeaway | Why |
|---|---|
| Default to L2 | Best balance of safety and effort for most features |
| Name things precisely | Ubiquitous language prevents entire categories of logic bugs |
| Trust the source of truth | Derived data is a bet on freshness |
| Linearize via infrastructure | Don't solve concurrency in app code if infrastructure can |
| Make metrics loud | "Is this broken?" in 5 seconds, not 5 minutes |
| Use the right tier | L1 when explicit, L5 only when the cost is justified |
