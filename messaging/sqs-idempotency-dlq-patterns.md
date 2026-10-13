# SQS Idempotency & Dead-Letter Queue Management

## Context

Building a contract management service on AWS ECS + SQS + DynamoDB. Messages can be delivered more than once, and downstream failures must not silently vanish.

## Idempotency Pattern

Store a dedupe key (message ID + operation hash) in DynamoDB with a TTL:

```typescript
const processMessage = async (message: SQSMessage) => {
  const dedupeKey = `${message.messageId}:${hashPayload(message.body)}`;

  const result = await dynamo
    .put({
      TableName: "IdempotencyStore",
      Item: {
        dedupeKey,
        processedAt: Date.now(),
        ttl: Math.floor(Date.now() / 1000) + 86400,
      },
      ConditionExpression: "attribute_not_exists(dedupeKey)",
    })
    .promise();

  // Only process if the put succeeded (i.e., key was new)
  await handleContractUpdate(message.body);
};
```

If `ConditionalCheckFailedException` is thrown, message was already processed — safe to ack and move on.

## Dead-Letter Queue (DLQ) Setup

```hcl
resource "aws_sqs_queue" "main" {
  name                       = "contract-processing"
  visibility_timeout_seconds = 300
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3
  })
}

resource "aws_sqs_queue" "dlq" {
  name = "contract-processing-dlq"
}
```

## DLQ Operations

- **CloudWatch Alarm** triggers when DLQ depth > 0
- **Lambda replays** messages in batches during off-peak hours
- **Alert routing** sends Slack notification with full message body for triage

## Lesson Learned

Initially I set `maxReceiveCount = 5`. Bad idea. A transient failure (e.g., DynamoDB throttling) retried 5 times over ~20 minutes, exhausting provisioned capacity. Dropped to 3 and saw faster downstream recovery.

## See Also

- `aws/fargate-spot-vs-on-demand-cost-optimization.md` (the ECS consumers processing this queue)
