# Building a Bulk-Data Modification Engine with FastAPI

## Context

The support team needed to update thousands of client records at once (e.g., bulk address changes after a merger). Uploading CSVs and processing row-by-row via the standard API would time out.

## Architecture

```
[Support UI] --upload CSV--> [FastAPI] --enqueue--> [SQS] --consume--> [Worker]
                                                              |
                                                              v
                                                        [DynamoDB / Postgres]
```

### FastAPI Endpoint

```python
@app.post("/bulk-modifications")
async def create_bulk_modification(
    file: UploadFile = File(...),
    current_user: User = Depends(get_current_user),
):
    rows = await parse_csv(file)
    job_id = str(uuid4())

    # Validate + enqueue, don't process inline
    await sqs.send_message(
        QueueUrl=BULK_QUEUE_URL,
        MessageBody=json.dumps({
            "jobId": job_id,
            "rows": rows,
            "requestedBy": current_user.id,
            "submittedAt": datetime.utcnow().isoformat(),
        }),
    )

    return {"jobId": job_id, "status": "enqueued", "estimatedRows": len(rows)}
```

### Worker Consumer (async batching)

```python
async def process_batch(messages: list[SQSMessage]):
    updates = []
    for msg in messages:
        updates.extend(msg.body["rows"])

    # Bulk upsert with ON CONFLICT DO UPDATE
    await db.execute(
        insert(Client)
        .values(updates)
        .on_conflict_do_update(
            index_elements=["client_id"],
            set_={"address": insert(Client).excluded.address}
        )
    )
```

## Key Decisions

1. **Never process in the request thread** — FastAPI workers are precious. Offload to SQS immediately.
2. **Batch at the worker** — SQS batch size of 10 messages × 500 rows each = 5,000 rows per PostgreSQL round-trip.
3. **Progress tracking** — A `jobs` table in Postgres tracked `PENDING → PROCESSING → COMPLETED | FAILED`. Support team polled `/bulk-modifications/{jobId}`.
4. **Validation gate** — Rejected the entire batch if >5% of rows failed schema validation, with a detailed error report.

## Impact

Support team efficiency increased by **~40%**. Previously they used manual SQL scripts or one-by-one UI edits. Now they upload, grab coffee, check the job status page.

## Lesson Learned

Python's `asyncio` + `asyncpg` is shockingly fast for bulk writes, but **connection pool exhaustion** is real. Default pool size of 10 is fine for API requests, not for a worker consuming 10 SQS messages concurrently. I bumped the worker pool to 50 and added `max_overflow=20`.

## See Also

- `messaging/sqs-idempotency-dlq-patterns.md` (the SQS layer powering this)
- `nodejs/event-driven-refactor-lessons.md` (the broader event-driven context)
