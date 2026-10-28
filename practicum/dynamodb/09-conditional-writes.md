# Lab 09 — Conditional Writes

## Goals
- Condition expressions prevent unintended overwrites
- Optimistic locking uses a version attribute to detect concurrent modifications
- `attribute_exists` and `attribute_not_exists` guard creates and updates
- The failure mode (`ConditionalCheckFailedException`) is handled correctly

---

## Why conditional writes?

Without conditions, two processes updating the same item simultaneously create a **last-writer-wins** race — one update silently overwrites the other. Condition expressions let you specify what must be true about the current item state before the write proceeds.

---

## Task 1 — Prevent overwriting an existing item

A table `config` exists (partition key: `configKey`, String).

A PutItem with condition `attribute_not_exists(configKey)` writes a new item:

```
configKey: "max-retries"   value: "3"   createdBy: "admin"
```

The write succeeds because no item with `configKey = "max-retries"` existed.

A second PutItem with the same condition and `configKey = "max-retries"` fails with `ConditionalCheckFailedException` — the item already exists. The existing item is unchanged.

Without the condition, the second PutItem would silently overwrite the first.

---

## Task 2 — Only update if item exists

An UpdateItem with condition `attribute_exists(configKey)` attempts to change `value` to `"5"` on `max-retries`.

This succeeds because the item exists.

The same UpdateItem is attempted on a non-existent key `min-retries`. It fails with `ConditionalCheckFailedException` — no item is created. This is the safe "update only, never create" pattern.

---

## Task 3 — Conditional value check

A table `inventory` exists (partition key: `sku`, String):

```
sku: "WIDGET-A"   stock: 10   reserved: 2
```

An UpdateItem decrements `stock` by 5 with the condition `stock >= :amount`:

```
UpdateExpression: SET stock = stock - :amount
ConditionExpression: stock >= :amount
ExpressionAttributeValues: {":amount": 5}
```

This succeeds — stock becomes 5.

A second attempt to decrement by 8 fails — `5 < 8`, so the condition is false. `stock` remains 5. This prevents negative stock without a read-before-write round trip.

---

## Task 4 — Optimistic locking

A table `documents` exists (partition key: `docId`, String):

```
docId: "doc-001"   content: "Initial draft"   version: 1
```

Optimistic locking works by including the current version in the condition, and incrementing it on every write:

**Read:** Fetch the item and note `version = 1`.

**Update (Process A):**
```
UpdateExpression: SET content = :newContent, version = version + :one
ConditionExpression: version = :expectedVersion
ExpressionAttributeValues: {
  ":newContent": "Updated by A",
  ":expectedVersion": 1,
  ":one": 1
}
```
Succeeds — version becomes 2.

**Concurrent update (Process B, also read version = 1):**
Same condition `version = :expectedVersion` with `:expectedVersion = 1`. Fails with `ConditionalCheckFailedException` — the version is now 2. Process B must re-read the item (getting version 2), merge its changes, and retry.

This pattern detects lost updates without database-level locks.

---

## Task 5 — Complex conditions

A table `loan-applications` exists (partition key: `applicationId`, String):

```
applicationId: "loan-001"   status: "pending"   amount: 5000   creditScore: 720
```

An UpdateItem approves the loan only when all conditions are met:

```
ConditionExpression: #status = :pending AND creditScore >= :minScore AND amount <= :maxAmount
ExpressionAttributeNames: {"#status": "status"}
ExpressionAttributeValues: {
  ":pending": "pending",
  ":minScore": 700,
  ":maxAmount": 10000
}
UpdateExpression: SET #status = :approved
```

The update succeeds for `loan-001` (score 720 ≥ 700, amount 5000 ≤ 10000).

A second application with `creditScore: 680` fails the condition — status remains `pending`.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Failed condition | Returns `ConditionalCheckFailedException` — item unchanged |
| `attribute_not_exists` | True when the attribute (or item) does not exist |
| `attribute_exists` | True when the attribute exists |
| Optimistic locking | Version attribute — compare-and-swap without locks |
| Condition cost | Conditions do not change RCU/WCU cost |
| Expression attribute names | Required for reserved words (e.g., `status`, `name`) |
| Transactions + conditions | Each operation in a TransactWriteItems can have its own condition |

---

## Clean up

Tables `config`, `inventory`, `documents`, and `loan-applications` are deleted.
