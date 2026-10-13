# Immutable Infrastructure for 100% Environment Parity

## Context

Provisioning production and staging AWS environments with Terraform. Goal: identical infrastructure, zero drift.

## The Pattern

Instead of mutating live resources (e.g., changing an ECS task definition in place), treat every deploy as a replacement:

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.service_name}-${var.environment}"
  container_definitions    = jsonencode(local.container_defs)
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory

  # Force new revision on every change
  lifecycle {
    create_before_destroy = true
  }
}
```

## Key Takeaways

- **No manual console changes** — if it can't be destroyed and recreated via Terraform, it shouldn't exist.
- **Blue/Green by default** — Fargate tasks + ALB target groups make swaps safe.
- **Variable-driven parity** — same module, different variables (`prod` vs `staging`).
- **State isolation** — separate S3 backends per environment prevent cross-contamination.

## Cost Angle

Using `create_before_destroy` with Fargate Spot in staging cut non-prod compute costs by ~60% while keeping infra identical.

## See Also

- `aws/fargate-spot-vs-on-demand-cost-optimization.md`
