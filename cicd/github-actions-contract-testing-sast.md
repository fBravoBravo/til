# GitHub Actions Pipeline: Contract Testing + SAST

## The Goal

Cut "time to market" by moving security and contract validation **left** — into the CI pipeline, before any code reaches staging.

## Pipeline Stages

```yaml
name: CI/CD Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci
      - run: npm run lint
      - run: npm run test:unit -- --coverage --coverageThreshold=80

  contract-test:
    runs-on: ubuntu-latest
    needs: lint-and-test
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:contract
        # Validates OpenAPI spec against real controller responses

  security-scan:
    runs-on: ubuntu-latest
    needs: lint-and-test
    steps:
      - uses: actions/checkout@v4
      - name: SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/owasp-top-ten
      - name: Dependency Audit
        run: npm audit --audit-level=moderate

  deploy:
    needs: [contract-test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Terraform Plan & Apply
        run: |
          terraform plan -var="image_tag=${{ github.sha }}"
          terraform apply -auto-approve
```

## The Impact

- **Contract tests** caught ~15 breaking changes in PR reviews before they hit staging
- **SAST** flagged a hardcoded secret in a feature branch before merge (Semgrep rule `detected-aws-access-key-id`)
- Combined effect: **~20% faster releases** because staging became a verification step, not a debugging step

## Key Learning

Running contract tests against a **real Docker Compose stack** (Postgres, localstack SQS, Redis) is slower than mocking, but catches serialization issues that mocks hide. We split them:
- Mocked unit tests: fast feedback
- Contract tests: pre-merge gate only
