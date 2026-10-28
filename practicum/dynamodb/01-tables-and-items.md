# Lab 01 — Tables and Items

## Goals
- A DynamoDB table exists with a meaningful primary key design
- The difference between a simple primary key and a composite primary key is clear
- Items with different attributes coexist in the same table (schema-less)
- The relationship between partition key, sort key, and data distribution is understood

---

## Core concepts

| Concept | Description |
|---------|-------------|
| **Table** | The top-level container — no joins, no foreign keys |
| **Partition key** | Determines which physical partition holds the item; must be unique when used alone |
| **Sort key** | Optional second dimension; partition key + sort key together must be unique |
| **Item** | A single row — a collection of attributes |
| **Attribute** | A name-value pair; each item can have different attributes |

---

## Task 1 — Simple primary key (partition key only)

A table named `users` exists with:
- Partition key: `userId` (String)
- Billing mode: On-demand

The following items exist in the table:

```
userId: "u-001"   name: "Alice"   email: "alice@example.com"   age: 30
userId: "u-002"   name: "Bob"     email: "bob@example.com"
userId: "u-003"   name: "Carol"   email: "carol@example.com"   role: "admin"   department: "eng"
```

Each item has a different set of attributes beyond `userId`. DynamoDB imposes no schema — only the key attribute (`userId`) is required.

**Observation:** Fetching item `u-001` returns only the attributes stored for that item. There is no `null` for `role` — the attribute simply does not exist.

---

## Task 2 — Composite primary key (partition key + sort key)

A table named `orders` exists with:
- Partition key: `customerId` (String)
- Sort key: `orderId` (String)
- Billing mode: On-demand

The following items exist:

```
customerId: "c-001"   orderId: "o-2024-001"   total: 49.99    status: "shipped"
customerId: "c-001"   orderId: "o-2024-002"   total: 120.00   status: "pending"
customerId: "c-001"   orderId: "o-2024-003"   total: 8.50     status: "delivered"
customerId: "c-002"   orderId: "o-2024-004"   total: 200.00   status: "shipped"
```

All orders for `c-001` live in the same partition. Querying by `customerId = "c-001"` returns all three of their orders efficiently — no scan needed.

**Observation:** Two items can share the same partition key as long as their sort keys differ. The combination `(customerId, orderId)` is globally unique.

---

## Task 3 — Key design exercise

A new table named `sessions` is needed to store user session tokens. The access patterns are:

1. Look up a session by its token (exact match)
2. List all active sessions for a given user

Two design options:

**Option A — Simple key:**
- Partition key: `sessionToken` (String)

**Option B — Composite key:**
- Partition key: `userId` (String)
- Sort key: `sessionToken` (String)

The table `sessions` is created using **Option B**.

Items are written for two users, each with two sessions. A query on `userId = "u-001"` returns both sessions for that user. A query on `userId = "u-001"` and `sessionToken = "tok-abc"` returns exactly one item.

**Option A cannot satisfy access pattern 2 without a full table scan.** Option B satisfies both patterns using Query alone.

---

## Task 4 — Observe item size limits

A table named `blob-test` exists (partition key: `id`, String).

An item is written where one attribute value is a string of exactly **400 KB** (DynamoDB's item size limit is 400 KB including attribute names and values).

A second write is attempted where the total item size exceeds 400 KB. The write is rejected with a `ValidationException: Item size has exceeded the maximum allowed size`.

---

## Key facts to remember

| Property | Value |
|----------|-------|
| Max item size | 400 KB |
| Partition key types | String, Number, Binary |
| Sort key types | String, Number, Binary |
| Schema enforcement | None — only key attributes are required |
| Same partition key, different sort key | Allowed (composite key tables) |
| Same partition key, same sort key | Not allowed — overwrites the existing item |

---

## Clean up

Tables `users`, `orders`, `sessions`, and `blob-test` are deleted.
