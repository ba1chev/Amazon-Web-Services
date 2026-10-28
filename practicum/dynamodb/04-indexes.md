# Lab 04 — Global and Local Secondary Indexes

## Goals
- A GSI enables efficient queries on non-key attributes
- An LSI provides an alternate sort key within the same partition
- The difference in consistency, key constraints, and cost between GSI and LSI is understood
- Index design follows access patterns, not the other way around

---

## GSI vs LSI

| Property | GSI | LSI |
|----------|-----|-----|
| Partition key | Any attribute | Must be the **same** as the table |
| Sort key | Any attribute (optional) | Different from the table's sort key |
| Creation | Anytime | Only at table creation |
| Consistency | Eventually consistent only | Strongly consistent supported |
| Capacity | Separate from the table | Shares table's capacity |
| Limit | 20 per table | 5 per table |

---

## Task 1 — GSI: query by a non-key attribute

A table `employees` exists with:
- Partition key: `employeeId` (String)
- Billing mode: On-demand

Items:

```
employeeId: "e-001"   name: "Alice"   department: "engineering"   city: "Berlin"
employeeId: "e-002"   name: "Bob"     department: "marketing"     city: "London"
employeeId: "e-003"   name: "Carol"   department: "engineering"   city: "Berlin"
employeeId: "e-004"   name: "Dave"    department: "engineering"   city: "Paris"
```

**Without a GSI**, finding all engineers requires a Scan with a filter on `department` — DynamoDB reads every item.

**A GSI named `department-index`** exists with:
- Partition key: `department` (String)
- Sort key: `employeeId` (String) — projected from the base table

A Query on `department-index` with `department = "engineering"` returns `e-001`, `e-003`, and `e-004` efficiently — no scan.

**Projection:** The GSI is configured with `KEYS_ONLY` projection. Only `department`, `employeeId`, and the table's primary key are stored in the index. Accessing `name` or `city` from a GSI result requires an additional `GetItem` on the base table.

A second GSI `city-department-index` exists with partition key `city` and sort key `department`. A Query on `city = "Berlin"` returns both Berlin employees. A Query on `city = "Berlin"` and `department = "engineering"` returns only `e-001` and `e-003`.

---

## Task 2 — GSI with sparse index

A table `tasks` exists (partition key: `taskId`, String) with these items:

```
taskId: "t-001"   title: "Fix bug"        status: "open"      assignee: "alice"
taskId: "t-002"   title: "Write docs"     status: "done"
taskId: "t-003"   title: "Deploy service" status: "open"      assignee: "bob"
taskId: "t-004"   title: "Code review"    status: "done"
```

A GSI `assignee-index` exists with partition key `assignee` (String).

Because only items with an `assignee` attribute are indexed, the GSI contains only `t-001` and `t-003`. Items without `assignee` are invisible to the GSI — this is a **sparse index**.

A Query on `assignee-index` with `assignee = "alice"` returns only `t-001`. Completed tasks (with no assignee) never appear in this index, reducing its size and cost.

---

## Task 3 — LSI: alternate sort key

A table `forum-posts` exists with:
- Partition key: `forumId` (String)
- Sort key: `postId` (String)
- An LSI named `score-index` with sort key `score` (Number)
- Both LSI and base table use `ALL` projection

Items:

```
forumId: "aws"   postId: "p-001"   title: "Lambda tips"      score: 142
forumId: "aws"   postId: "p-002"   title: "S3 pricing"       score: 87
forumId: "aws"   postId: "p-003"   title: "DynamoDB indexes" score: 310
forumId: "aws"   postId: "p-004"   title: "EC2 spot guide"   score: 55
```

A Query on the **base table** with `forumId = "aws"` returns posts ordered by `postId` (alphabetically).

A Query on **`score-index`** with `forumId = "aws"` and `ScanIndexForward: false` returns posts ordered by `score` descending — highest-scored posts first. This is the "top posts in forum" access pattern with no scan.

A Query on `score-index` with `forumId = "aws"` and `score > 100` returns only `p-001` and `p-003`.

**Consistency:** Because this is an LSI, the Query can use `ConsistentRead: true` and will always reflect the latest writes. This is not possible on a GSI.

---

## Task 4 — Index write amplification

An item is written to `employees` that has both `department` and `city` attributes. DynamoDB writes to:
1. The base table
2. `department-index`
3. `city-department-index`

Each of these consumes WCUs independently. A single write to the base table results in 3 total writes. This is **write amplification** — the cost of maintaining indexes.

An item with no `department` attribute is written. It is stored in the base table but does **not** appear in `department-index` — DynamoDB only indexes items that have the index's key attribute.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| GSI consistency | Eventually consistent only |
| LSI consistency | Strongly consistent supported |
| LSI creation | Only at table creation — cannot be added later |
| GSI creation | Can be added to an existing table |
| Sparse index | Items missing the index key attribute are not indexed |
| Write amplification | Each GSI/LSI adds write cost for every item write |
| GSI throttling | GSIs have their own capacity — a throttled GSI does not block the base table |

---

## Clean up

Tables `employees`, `tasks`, and `forum-posts` are deleted.
