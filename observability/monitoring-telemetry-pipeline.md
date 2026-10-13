# Building an Observability Stack That Actually Helps During Incidents

## Context

Pre-refactor, our services had CloudWatch alarms and some scattered logs. When something broke, finding the root cause took 20–45 minutes. Post-refactor, mean time to detect (MTTD) and resolve (MTTR) dropped dramatically.

## What I Built

### 1. Structured Logging Everywhere

Unified JSON format across all services:

```typescript
logger.info("Contract processed", {
  traceId,
  spanId,
  service: "contract-service",
  event: "contract.activated",
  contractId,
  durationMs,
  retryCount,
});
```

Why this shape? It indexes perfectly in CloudWatch Logs Insights:

```sql
fields @timestamp, durationMs, retryCount
| filter service = "contract-service" and event = "contract.activated"
| stats avg(durationMs) by bin(5m)
```

### 2. Distributed Tracing (AWS X-Ray)

Every request gets a `traceId` at the ALB, propagated through headers to all downstream services. X-Ray service map shows the call graph and latency per hop.

Key integration: X-Ray SDK in NestJS + AWS SDKv3 instrumented automatically:

```typescript
const client = new SQSClient({
  region: "eu-west-1",
});
// SDKv3 middleware attaches trace headers automatically
```

### 3. Custom Metrics via CloudWatch Embedded Metric Format (EMF)

Instead of relying on log-based metrics (slow, expensive), services emit EMF logs that CloudWatch parses into metrics automatically:

```typescript
const emf = {
  _aws: {
    Timestamp: Date.now(),
    CloudWatchMetrics: [
      {
        Namespace: "ContractService",
        Dimensions: [["Environment", "ServiceName"]],
        Metrics: [{ Name: "ProcessingDuration", Unit: "Milliseconds" }],
      },
    ],
  },
  Environment: "production",
  ServiceName: "contract-service",
  ProcessingDuration: duration,
};
console.log(JSON.stringify(emf));
```

This costs ~1/10th of what PutMetricData calls would cost at our volume.

### 4. Alerting Hierarchy

| Severity | Response Time | Channel | Rule Example |
|----------|---------------|---------|--------------|
| P1 | 5 min | Page (PagerDuty) | DLQ depth > 0 for > 2 min |
| P2 | 15 min | Slack #alerts | Error rate > 1% for 5 min |
| P3 | 1 hour | Slack #observability | Latency p99 > 500ms for 20 min |

## The Unexpected Win

Once the telemetry was solid, we caught a **gradual memory leak** in a Python service via CloudWatch Container Insights. Memory usage grew 2MB/hour. Without the baselines, it would have caused an outage 3 weeks later.

## See Also

- `cicd/github-actions-contract-testing-sast.md` (catches issues before they reach this pipeline)
- `messaging/sqs-idempotency-dlq-patterns.md` (DLQ alerting referenced above)
