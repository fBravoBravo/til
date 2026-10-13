# Using LLMs to Generate Swagger/OpenAPI Documentation

## The Pain Point

Our middleware layer had dozens of internal endpoints. Docs were perpetually out of sync with code. Maintaining them manually meant:
- Developer friction
- Stale/incorrect consumer integrations
- Support tickets asking "what does this field do?"

## The Approach

I built a pipeline that feeds controller source code + JSDoc comments to an LLM and generates a draft OpenAPI spec, then gates merges on human review.

## Pipeline (GitHub Actions)

```yaml
- name: Generate OpenAPI spec
  run: |
    npx ts-node scripts/generate-openapi.ts \
      --input ./src/controllers \
      --output ./docs/openapi.yaml \
      --model gpt-4o
```

The script walks controllers, extracts:
- Route decorators (`@Controller`, `@Get`, `@Post`)
- Method signatures and DTOs
- JSDoc descriptions
- Error response patterns

Then asks the LLM to synthesize a complete `openapi.yaml`.

## Prompt Engineering Trick

I found that giving the LLM a **partial spec** (already-correct endpoints from the prior run) as context dramatically reduced hallucination. The model "fills in" only new/changed routes instead of regenerating everything from scratch.

## Coverage Integration

The generated spec feeds into our contract test suite via `Dredd` and `Schemathesis`. If the spec is wrong, tests fail. If tests pass, the spec is validated.

## Result

- Spec accuracy went from ~40% manual coverage to **consistently current**
- New endpoint docs are draft-ready before the PR is merged
- Consumer teams stopped pinging us on Slack for field meanings

## Caution

LLMs occasionally infer types incorrectly when TypeScript types are implicit (e.g., `any` or overloaded methods). We added a strict `noImplicitAny` lint rule as a prerequisite.
