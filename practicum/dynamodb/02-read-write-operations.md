# Lab 02 — Read and Write Operations

## Goals
- Individual items are created, read, updated, and deleted
- The difference between `PutItem` (full replace) and `UpdateItem` (partial update) is clear
- `GetItem` returns a strongly consistent or eventually consistent read
- Batch operations reduce round trips for multiple items

---

## Operations reference

| Operation | What it does |
|-----------|-------------|
| `PutItem` | Creates or fully replaces an item |
| `GetItem` | Reads one item by its exact primary key |
| `UpdateItem` | Adds, modifies, or removes specific attributes without touching others |
| `DeleteItem` | Removes one item by its exact primary key |
| `BatchWriteItem` | Up to 25 PutItem or DeleteItem in one request |
| `BatchGetItem` | Up to 100 GetItem across one or more tables |

---

## Task 1 — PutItem vs UpdateItem

A table named `products` exists with partition key `productId` (String).

**PutItem** — the following item is written:

```json
{
  "productId": "p-001",
  "name": "Laptop",
  "price": 999.99,
  "stock": 50,
  "category": "electronics"
}
```

**PutItem again with fewer attributes:**

```json
{
  "productId": "p-001",
  "name": "Laptop Pro",
  "price": 1199.99
}
```

After the second PutItem, a GetItem on `p-001` returns only `name` and `price`. The `stock` and `category` attributes are gone — PutItem replaced the entire item.

**UpdateItem** — starting again from the full item, an UpdateItem sets only `price` to `1299.99`:

```
UpdateExpression: SET price = :newPrice
ExpressionAttributeValues: {":newPrice": 1299.99}
```

After this UpdateItem, `stock` and `category` are still present. Only `price` changed.

---

## Task 2 — UpdateItem expressions

Starting from item `p-001` with attributes `name`, `price`, `stock`, `category`, the following updates are applied one at a time:

**Add a new attribute:**
```
SET description = :desc
```
The item now has a `description` attribute that did not exist before.

**Increment a number:**
```
SET stock = stock + :delta
ExpressionAttributeValues: {":delta": 10}
```
`stock` increases by 10 without reading the current value first.

**Remove an attribute:**
```
REMOVE description
```
The `description` attribute is gone from the item.

**Add to a string set:**
```
SET tags = if_not_exists(tags, :empty) 
ADD tags :newTag
```
Or using the ADD action directly on a set attribute.

---

## Task 3 — Consistent reads

A table `counters` exists with partition key `name` (String). An item `{"name": "pageViews", "value": 0}` is written.

**Eventually consistent read (default):**
A GetItem returns the item. In a distributed system, a replica may not yet have the latest write — DynamoDB may return a slightly stale value. This uses **0.5 RCU** per 4 KB.

**Strongly consistent read:**
A GetItem with `ConsistentRead: true` always returns the latest committed write. This uses **1 RCU** per 4 KB.

In the console, a strongly consistent read on the item returns the current value immediately after a write. Note that strongly consistent reads are not supported on Global Secondary Indexes.

---

## Task 4 — BatchWriteItem

A table `inventory` exists (partition key: `sku`, String).

A single BatchWriteItem request writes the following 4 items simultaneously:

```
sku: "SKU-A"  name: "Widget A"  qty: 100
sku: "SKU-B"  name: "Widget B"  qty: 250
sku: "SKU-C"  name: "Widget C"  qty: 75
sku: "SKU-D"  name: "Widget D"  qty: 0
```

A second BatchWriteItem deletes `SKU-D` and writes a new item `SKU-E` in the same request.

**Observation:** BatchWriteItem is not atomic — some items may succeed while others fail. It does not support condition expressions. For atomic multi-item writes, see Lab 07 (Transactions).

---

## Task 5 — BatchGetItem

A single BatchGetItem request retrieves `SKU-A`, `SKU-B`, and `SKU-C` from `inventory` in one API call.

The response contains all three items. If one key does not exist, the response simply omits it — no error is thrown.

**Observation:** BatchGetItem can fetch from multiple tables in one call. Unprocessed keys (throttled items) appear in `UnprocessedKeys` — the caller is responsible for retrying them.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| PutItem | Replaces the entire item — existing attributes not in the new item are lost |
| UpdateItem | Modifies specific attributes — other attributes are untouched |
| Eventually consistent read | 0.5 RCU per 4 KB — default for all reads |
| Strongly consistent read | 1 RCU per 4 KB — not available on GSIs |
| BatchWriteItem limit | 25 operations per request |
| BatchGetItem limit | 100 items per request |
| BatchWriteItem atomicity | Not atomic — partial success is possible |

---

## Clean up

Tables `products`, `counters`, and `inventory` are deleted.
