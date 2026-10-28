# Lab 02 — Triggers and Event Sources

## Goals
- A Lambda function is invoked via API Gateway, S3, and DynamoDB Streams
- You can read and interpret the event payload for each source
- You understand the difference between synchronous and asynchronous invocation

---

## Invocation models — quick reference

| Source | Model | Retries on error |
|--------|-------|-----------------|
| API Gateway | Synchronous | None — caller receives the error |
| S3 | Asynchronous | 2 automatic retries |
| DynamoDB Streams | Polling (event source mapping) | Until stream expiry or success |
| SQS | Polling (event source mapping) | Configurable |
| SNS | Asynchronous | 2 automatic retries |

---

## Task 1 — API Gateway trigger (synchronous)

A Lambda function named `api-lab` exists with an API Gateway (REST API) trigger.

The handler reads `httpMethod`, `path`, and `body` from the event and returns them in the response:

```python
import json

def lambda_handler(event, context):
    method = event["httpMethod"]
    path   = event["path"]
    body   = json.loads(event.get("body") or "{}")
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"method": method, "path": path, "received": body})
    }
```

A POST request to the API endpoint returns a 200 response with the echoed values.

**Key insight:** API Gateway waits for the Lambda response synchronously. If the function errors, the HTTP caller receives the error directly — there are no automatic retries.

---

## Task 2 — S3 trigger (asynchronous)

A Lambda function named `s3-lab` exists. An S3 bucket is configured to invoke the function on every object upload (`s3:ObjectCreated:*`).

The handler logs the bucket name, object key, and size of every uploaded object:

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key    = record["s3"]["object"]["key"]
        size   = record["s3"]["object"].get("size", "?")
        print(f"New object: s3://{bucket}/{key} ({size} bytes)")
```

After uploading a file to the bucket, the log entry appears in CloudWatch Logs.

The execution role has `s3:GetObject` permission on the bucket.

**Key insight:** S3 invokes Lambda asynchronously — S3 does not wait for the function to finish. If the function fails, Lambda retries up to 2 times automatically. The caller (whoever uploaded the file) never sees the Lambda error.

---

## Task 3 — DynamoDB Streams (event source mapping)

A DynamoDB table named `orders` (partition key: `id`) exists with **DynamoDB Streams** enabled using view type **New and old images**.

A Lambda function named `dynamo-lab` is connected to the stream via an event source mapping with:
- Batch size: 10
- Starting position: Latest

The handler logs the event name (`INSERT`, `MODIFY`, `REMOVE`) along with the new and old images:

```python
import json

def lambda_handler(event, context):
    for record in event["Records"]:
        event_name = record["eventName"]
        new_image  = record["dynamodb"].get("NewImage", {})
        old_image  = record["dynamodb"].get("OldImage", {})
        print(f"Event: {event_name}")
        print(f"New:   {json.dumps(new_image)}")
        print(f"Old:   {json.dumps(old_image)}")
```

The execution role has the `AWSLambdaDynamoDBExecutionRole` managed policy attached.

After writing an item to the `orders` table, the INSERT event appears in CloudWatch. After updating that item, the MODIFY event appears with both the old and new image.

**Key insight:** This is a **polling** model — Lambda polls the stream on your behalf. The event source mapping is the bridge between the stream and the function. If the function fails, the same batch is retried until it succeeds or the records expire (up to 7 days).

---

## Clean up

Functions `api-lab`, `s3-lab`, and `dynamo-lab` are deleted.
The DynamoDB `orders` table is deleted.
The S3 bucket notification is removed.
