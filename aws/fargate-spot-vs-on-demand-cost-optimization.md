# Fargate Spot vs On-Demand: When to Use Which

## The Numbers

| Environment | Strategy | Savings |
|-------------|----------|---------|
| `staging`   | Fargate Spot | ~70% |
| `prod`      | On-Demand (with Spot fallback) | ~30% on non-critical tasks |

## The Trade-off

- **Spot tasks** can be interrupted by AWS with a 2-minute warning.
- **On-Demand** is guaranteed but 3–4x the price.

## Implementation

Use a capacity provider strategy in ECS:

```hcl
capacity_provider_strategy {
  capacity_provider = "FARGATE_SPOT"
  weight            = var.environment == "prod" ? 1 : 4
}

capacity_provider_strategy {
  capacity_provider = "FARGATE"
  weight            = var.environment == "prod" ? 3 : 1
  base              = var.environment == "prod" ? 1 : 0
}
```

In **prod**, I keep a `base = 1` on Fargate so at least one task is always guaranteed running, while Spot handles burst.

## When NOT to use Spot

- Long-running batch jobs that can't checkpoint
- Real-time WebSocket services where interruption = visible user impact
- Anything where retry logic is expensive or impossible

## Lesson Learned

Saving money is easy. Saving money *without introducing env drift* requires the same task definition on both capacity providers — something only possible if infra is fully Terraform-managed.
