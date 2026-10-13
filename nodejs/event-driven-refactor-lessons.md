# Lessons from Refactoring a Monolith to Event-Driven Architecture

## Context

Led a refactor of legacy services (tightly coupled synchronous calls) into an event-driven model using SQS + SNS. Goal: decouple domains, improve resilience, enable independent scaling.

## The Win

- **30% reduction** in error rates
- **20% performance boost** (fewer blocking calls, more parallel processing)

## Patterns Used

### 1. Transactional Outbox

Instead of publishing events inline with DB writes (risk: message sent, DB rolled back), I used an **outbox table**:

```typescript
// In the same transaction as the business write
await db.transaction(async (trx) => {
  await trx("contracts").insert(newContract);
  await trx("outbox").insert({
    topic: "contract.created",
    payload: JSON.stringify(newContract),
    created_at: new Date(),
  });
});
```

A background poller reads `outbox` and publishes to SNS. Guarantees at-least-once delivery without distributed transactions.

### 2. Event Schema Evolution

Enforced backward compatibility using **JSON Schema** in a central registry. Every event consumer declares the schema version it expects. Producers can't break consumers without a major version bump.

### 3. Saga Pattern (for cross-service consistency)

A "contract activation" flow touches 3 services. Instead of distributed transactions:
1. Service A emits `ContractRequested`
2. Service B processes, emits `ContractValidated`
3. Service C processes, emits `ContractActivated`

If any step fails, a compensating event (`ContractActivationFailed`) triggers rollback actions in reverse order.

## What I Got Wrong Initially

I started with **choreography** (every service reacts to events independently). As the mesh grew to 8+ services, tracing a single request required checking 6 logs.

Switched to **orchestration via a lightweight saga coordinator** (a single NestJS module with a state machine). Debugging became trivial — one log stream shows the whole flow.

## Lesson

Event-driven is not "fire and forget." Without idempotency, DLQs, and observability, it becomes a distributed debugging nightmare. See `messaging/sqs-idempotency-dlq-patterns.md` for the resilience layer.
