# Lab 01 — First Lambda Function

## Goals
- A Lambda function exists and can be invoked
- The handler correctly reads from the `event` object
- You can read the function's logs in CloudWatch

---

## Task 1 — Basic function

A Python 3.12 Lambda function named `hello-lab` exists with the following handler:

```python
import json

def lambda_handler(event, context):
    name = event.get("name", "World")
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

When invoked with `{"name": "Alice"}`, the response body contains `Hello, Alice!`.

---

## Task 2 — Context object

The function logs the following fields to CloudWatch on every invocation:

- `context.aws_request_id`
- `context.get_remaining_time_in_millis()`
- `context.memory_limit_in_mb`
- `context.function_version`

After invoking the function, the log stream in CloudWatch Logs shows these values.

```python
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info("Request ID: %s", context.aws_request_id)
    logger.info("Remaining time (ms): %d", context.get_remaining_time_in_millis())
    logger.info("Memory limit (MB): %s", context.memory_limit_in_mb)
    logger.info("Function version: %s", context.function_version)
    return {"statusCode": 200}
```

---

## Task 3 — Timeout and memory

The function's timeout is set to **30 seconds** and memory to **256 MB**.

After invoking, the CloudWatch log entry shows `Billed Duration` that reflects the actual execution time, not the full timeout.

---

## Task 4 — Observe a cold start

The function's code includes a `time.sleep(2)` call. After deploying a new version, the first invocation's CloudWatch log entry contains an `Init Duration` line. Subsequent invocations on a warm environment do not.

---

## Task 5 — Force a timeout

The function's timeout is set to **5 seconds**. The handler sleeps for **60 seconds**. When invoked, the function fails with a `Task timed out` error visible in the execution result and CloudWatch.

After confirming the behavior, restore the timeout and fix the code.

---

## Key facts to remember

| Property | Value |
|----------|-------|
| Default timeout | 3 seconds |
| Maximum timeout | 900 seconds (15 min) |
| Memory range | 128 MB – 10,240 MB |
| CPU scales with | memory (full vCPU ≈ 1,769 MB) |
| Max package size (direct upload) | 50 MB zipped |
| Max package size (via S3) | 250 MB unzipped |
| `Init Duration` appears | Only on cold starts |

---

## Clean up

The `hello-lab` function is deleted.
