# Lab 08 — Error Handling

## Goals
- Retry behavior differs between synchronous and asynchronous invocations
- Failed asynchronous events are captured in a Dead Letter Queue (DLQ)
- Lambda Destinations route both successes and failures with full context
- Event source mapping failures (DynamoDB Streams) can be isolated without stalling the stream

---

## Retry behavior — quick reference

| Invocation type | Retries on error | Where failed events go |
|-----------------|-----------------|------------------------|
| Synchronous (API GW) | 0 — caller gets the error immediately | Caller must handle |
| Asynchronous (S3, SNS) | Up to 2 (configurable 0–2) | DLQ or On-failure Destination |
| Event source mapping (SQS) | Until visibility timeout expires | DLQ on the SQS queue |
| Event source mapping (Streams) | Until stream expiry | On-failure Destination |

---

## Task 1 — Synchronous error (no automatic retry)

A function `error-lab` (Python 3.12) raises an exception when `event.fail` is true:

```python
def lambda_handler(event, context):
    if event.get("fail"):
        raise ValueError("Intentional error!")
    return {"ok": True}
```

When invoked with `{"fail": true}`, the execution result panel shows the error and traceback.

When invoked again immediately with the same payload, the error repeats. Lambda made no automatic retry — the error was returned to the caller on the first attempt.

---

## Task 2 — Asynchronous DLQ

An SQS Standard queue named `lambda-dlq` exists.

`error-lab` is configured for asynchronous invocation with:
- Maximum age of event: **60 seconds**
- Retry attempts: **1**
- Dead-letter queue: `lambda-dlq`

The execution role has `sqs:SendMessage` on `lambda-dlq`.

When the function is invoked asynchronously with `{"fail": true}`, Lambda retries once (2 total attempts) before the event ages out. The failed event then appears in `lambda-dlq`.

The SQS message body contains the original event. Message attributes include the error reason and the function ARN.

---

## Task 3 — Lambda Destinations

Two SQS queues exist: `lambda-success-queue` and `lambda-failure-queue`.

`error-lab` has Destinations configured:
- **On success** → `lambda-success-queue`
- **On failure** → `lambda-failure-queue`

The execution role has `sqs:SendMessage` on both queues.

After asynchronous invocations with both success and failure payloads:

- `lambda-success-queue` contains a message with the function response, request ID, and original event.
- `lambda-failure-queue` contains a message with the error details, stack trace, request ID, and original event.

**Destinations vs DLQ:**
- A DLQ receives only the original event on failure.
- A Destination receives the full execution context (event + response/error + metadata) for both outcomes.

---

## Task 4 — Stream mapping error handling

A DynamoDB stream event source mapping (from Lab 02) is configured with:
- **Bisect batch on function error:** enabled
- **On-failure destination:** `lambda-failure-queue`
- **Maximum retry attempts:** 2

The function intentionally raises an exception for a specific item ID:

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        item_id = record["dynamodb"].get("NewImage", {}).get("id", {}).get("S")
        if item_id == "bad-item":
            raise ValueError(f"Cannot process item: {item_id}")
        print(f"Processed: {item_id}")
```

After writing a `bad-item` to the DynamoDB table alongside valid items:

1. Lambda fails on the batch containing `bad-item`.
2. With bisect-on-error enabled, Lambda splits the batch and retries each half.
3. The half containing only `bad-item` fails again after 2 retries and is sent to `lambda-failure-queue`.
4. The half with valid items succeeds.
5. The stream is not stalled — processing continues for subsequent records.

Without bisect-on-error, the entire batch would retry until the record expires (7 days), blocking all subsequent stream records.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Async max retry attempts | 0–2 (default 2) |
| Async max event age | 60 s – 6 h (default 6 h) |
| DLQ vs Destination | DLQ = failure only, original event only; Destination = both outcomes, full context |
| Execution role for DLQ | Needs `sqs:SendMessage` on the DLQ ARN |
| Bisect-on-error | Isolates the failing record without stalling the stream |
| SQS DLQ | Configured on the SQS queue itself, not on Lambda |

---

## Clean up

`error-lab` is deleted. SQS queues `lambda-dlq`, `lambda-success-queue`, and `lambda-failure-queue` are deleted.
