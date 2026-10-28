# Lab 06 — DynamoDB Streams

## Goals
- A DynamoDB Stream captures every item-level change in order
- The stream view type controls how much data is captured per change
- A Lambda function processes stream records as they arrive
- The relationship between shards, sequence numbers, and ordering is understood

---

## Stream view types

| View type | What each record contains |
|-----------|--------------------------|
| `KEYS_ONLY` | Only the partition key and sort key of the changed item |
| `NEW_IMAGE` | The item's full state after the change |
| `OLD_IMAGE` | The item's full state before the change |
| `NEW_AND_OLD_IMAGES` | Both the before and after state |

---

## Task 1 — Enable a stream and observe raw records

A table `audit-log` exists (partition key: `entityId`, String; sort key: `timestamp`, String) with streams enabled using **NEW_AND_OLD_IMAGES**.

The following operations are performed in order:

1. A new item is written: `entityId: "doc-001", timestamp: "2024-01-10T10:00:00Z", status: "draft"`
2. The item is updated: `status` changes to `"published"`
3. The item is deleted

The stream contains exactly 3 records. Inspecting each record in the DynamoDB Streams console (or via a Lambda trigger):

- Record 1: `eventName: "INSERT"`, `NewImage` contains the full item, `OldImage` is absent
- Record 2: `eventName: "MODIFY"`, `NewImage` has `status: "published"`, `OldImage` has `status: "draft"`
- Record 3: `eventName: "REMOVE"`, `OldImage` contains the last state of the item, `NewImage` is absent

---

## Task 2 — Lambda processing stream records

A Lambda function `stream-processor` reads from `audit-log`'s stream with:
- Event source mapping batch size: 5
- Starting position: `TRIM_HORIZON` (reads from the oldest available record)

The handler builds an audit trail from each stream record:

```python
import json

def lambda_handler(event, context):
    for record in event["Records"]:
        event_name = record["eventName"]
        keys       = record["dynamodb"]["Keys"]
        new_image  = record["dynamodb"].get("NewImage", {})
        old_image  = record["dynamodb"].get("OldImage", {})

        entity_id  = keys["entityId"]["S"]

        if event_name == "INSERT":
            print(f"Created: {entity_id}")
        elif event_name == "MODIFY":
            old_status = old_image.get("status", {}).get("S", "?")
            new_status = new_image.get("status", {}).get("S", "?")
            print(f"Updated: {entity_id} — {old_status} → {new_status}")
        elif event_name == "REMOVE":
            print(f"Deleted: {entity_id}")
```

After performing the 3 operations above, CloudWatch Logs shows all three audit entries in the correct order.

---

## Task 3 — Stream ordering guarantees

A table `ordered-events` exists (partition key: `userId`, String; sort key: `eventId`, String) with streams enabled.

Ten items are written for `userId = "u-001"` in quick succession.

**Within a shard**, records for the same partition key arrive in the order they were written. The sequence numbers in `record["dynamodb"]["SequenceNumber"]` are monotonically increasing for items in the same shard.

**Across shards**, ordering is not guaranteed — but items with the same partition key always land in the same shard, so ordering is preserved per partition key.

**Observation:** Stream records for `u-001` arrive in the Lambda function in the exact write order. If a second user `u-002` writes simultaneously, their records may be interleaved across batches but are still correctly ordered relative to each other.

---

## Task 4 — Stream retention

Streams retain records for **24 hours**. After 24 hours, records are no longer available.

The `TRIM_HORIZON` starting position reads from the oldest available record (up to 24 h ago). The `LATEST` starting position reads only records written after the event source mapping was created.

A Lambda consumer that stops processing (e.g., function errors) must recover within 24 hours before records are lost.

---

## Task 5 — KEYS_ONLY stream to track deletes

A table `sessions` exists with streams enabled using **KEYS_ONLY**.

When a session item is deleted, the stream record contains only the key (`sessionId`) — not the full item. This is enough to know which session was deleted and remove it from a secondary cache.

A Lambda function listening to this stream:

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        if record["eventName"] == "REMOVE":
            session_id = record["dynamodb"]["Keys"]["sessionId"]["S"]
            print(f"Session expired: {session_id}")
            # Invalidate cache entry for session_id
```

`KEYS_ONLY` uses the least storage and produces the smallest stream records — appropriate when only the key is needed downstream.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Stream retention | 24 hours |
| Ordering guarantee | Per partition key, within a shard |
| Cross-partition ordering | Not guaranteed |
| Starting positions | `TRIM_HORIZON` (oldest), `LATEST`, `AT_SEQUENCE_NUMBER` |
| Lambda batch size | 1–10,000 records |
| Failed batch | Retried until records expire (24 h) unless bisect-on-error or max retries configured |
| Streams cost | Additional charge on top of table read/write costs |

---

## Clean up

Tables `audit-log`, `ordered-events`, and `sessions` are deleted.
Lambda function `stream-processor` and its event source mapping are deleted.
