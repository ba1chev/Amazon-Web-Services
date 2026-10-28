# Lab 08 — Time To Live (TTL)

## Goals
- TTL is configured on a table and items expire automatically
- The TTL attribute stores a Unix timestamp in seconds
- Expired items are not immediately deleted — the behavior is understood
- Streams capture TTL deletions differently from explicit DeleteItem calls

---

## How TTL works

You designate one attribute as the TTL attribute (must be a Number storing a Unix epoch timestamp in seconds). DynamoDB scans for expired items in the background and deletes them — typically within **48 hours** after the expiry time.

TTL deletions are **free** — they do not consume WCUs.

---

## Task 1 — Enable TTL on a table

A table `sessions` exists:
- Partition key: `sessionId` (String)
- TTL attribute: `expiresAt` (Number)

The following items exist. `expiresAt` values are Unix timestamps:

```
sessionId: "sess-001"   userId: "u-001"   expiresAt: <now + 60 seconds>
sessionId: "sess-002"   userId: "u-002"   expiresAt: <now + 300 seconds>
sessionId: "sess-003"   userId: "u-003"   (no expiresAt attribute)
```

Items with no `expiresAt` attribute are never expired by TTL.

**Observation:** `sess-001` disappears from the table within a few minutes of its expiry time (the background sweep runs periodically). `sess-003` remains indefinitely.

To compute a timestamp 60 seconds in the future in Python:
```python
import time
expires_at = int(time.time()) + 60
```

---

## Task 2 — Querying before and after expiry

A GetItem on `sess-001` **before** its `expiresAt` timestamp returns the item normally.

A GetItem on `sess-001` **at or shortly after** its `expiresAt` timestamp may still return the item — DynamoDB does not guarantee instant deletion. Items are logically expired but physically removed later.

**Important:** Expired items can still be read for up to 48 hours after their `expiresAt` time. Applications that need strict expiry must also check the `expiresAt` attribute in their own code:

```python
import time

def is_valid(item):
    expires_at = item.get("expiresAt", {}).get("N")
    if expires_at and int(expires_at) < time.time():
        return False
    return True
```

---

## Task 3 — Sliding expiry (refresh on access)

A table `api-tokens` exists (partition key: `token`, String; TTL attribute: `expiresAt`).

An item is written with `expiresAt = now + 3600` (1 hour from now).

Each time the token is used, an UpdateItem extends the TTL:

```
UpdateExpression: SET expiresAt = :newExpiry
ExpressionAttributeValues: {":newExpiry": <now + 3600>}
```

After each access, the expiry resets to 1 hour from the current time. An inactive token expires naturally without any explicit deletion.

---

## Task 4 — TTL and Streams

A table `ttl-stream-test` exists with TTL enabled on `expiresAt` and DynamoDB Streams enabled with **NEW_AND_OLD_IMAGES**.

An item with a past `expiresAt` is written. When DynamoDB's background process deletes it, a stream record appears with:
- `eventName: "REMOVE"`
- `userIdentity.type: "Service"` and `userIdentity.principalId: "dynamodb.amazonaws.com"`

This distinguishes a TTL deletion from an explicit `DeleteItem` call, which has no `userIdentity` field.

A Lambda function consuming this stream can skip TTL deletions if they should not trigger downstream processing:

```python
def lambda_handler(event, context):
    for record in event["Records"]:
        if record["eventName"] == "REMOVE":
            identity = record.get("userIdentity", {})
            if identity.get("type") == "Service":
                print("TTL deletion — skipping")
                continue
            print("Explicit deletion — processing")
```

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| TTL attribute type | Number — Unix epoch in **seconds** |
| Deletion timing | Background sweep — up to 48 h after expiry |
| TTL delete cost | Free — no WCU consumed |
| Items without TTL attr | Never expired automatically |
| Expired items in queries | May still appear — always validate client-side if strict |
| Stream record | `userIdentity.type = "Service"` identifies TTL deletions |
| TTL + GSI | Expired items are removed from GSIs when the base table item is deleted |

---

## Clean up

Tables `sessions`, `api-tokens`, and `ttl-stream-test` are deleted.
Lambda function consuming the stream is deleted.
