# Lab 03 — Queries and Scans

## Goals
- Query retrieves items efficiently using the partition key
- Scan reads every item in the table and is understood to be expensive
- Filter expressions reduce the response payload but not the read cost
- Projection expressions limit which attributes are returned

---

## Query vs Scan

| | Query | Scan |
|-|-------|------|
| Requires | Partition key value (exact) | Nothing — reads everything |
| Sort key | Optional — can filter or sort | N/A |
| Read cost | Only items in the partition | Every item in the table |
| Use case | Known access pattern | Ad-hoc, infrequent, small tables |

**Rule of thumb:** If you find yourself relying on Scan, the table's key design or index strategy should be reconsidered.

---

## Task 1 — Basic Query

A table `events` exists with:
- Partition key: `userId` (String)
- Sort key: `timestamp` (String, ISO-8601 format)

The following items exist:

```
userId: "u-001"   timestamp: "2024-01-10T08:00:00Z"   type: "login"
userId: "u-001"   timestamp: "2024-01-10T09:30:00Z"   type: "purchase"   amount: 49.99
userId: "u-001"   timestamp: "2024-01-11T07:45:00Z"   type: "login"
userId: "u-001"   timestamp: "2024-01-12T14:00:00Z"   type: "logout"
userId: "u-002"   timestamp: "2024-01-10T10:00:00Z"   type: "login"
userId: "u-002"   timestamp: "2024-01-10T11:00:00Z"   type: "purchase"   amount: 200.00
```

A Query with `userId = "u-001"` returns all 4 items for that user, ordered by `timestamp` ascending. DynamoDB always returns items within a partition sorted by the sort key.

A Query with `userId = "u-001"` and `ScanIndexForward: false` returns the same 4 items in descending order (most recent first).

---

## Task 2 — Sort key conditions

Using the same `events` table, a Query with:
- `userId = "u-001"`
- `timestamp BETWEEN "2024-01-10T00:00:00Z" AND "2024-01-10T23:59:59Z"`

Returns only the 2 events from January 10th for user `u-001`.

A Query with:
- `userId = "u-001"`
- `timestamp begins_with "2024-01-11"`

Returns the single event from January 11th.

**Observation:** Sort key conditions (`=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `begins_with`) are applied server-side before data is returned — they reduce read cost. This is different from a filter expression.

---

## Task 3 — Filter expressions

A Query with `userId = "u-001"` and a filter expression `type = "login"` returns 2 items.

**Observation:** DynamoDB first reads all 4 items in the partition (consuming RCUs for all 4), then discards non-matching items. The filter expression reduces the response payload — it does not reduce the read cost. Check the `ScannedCount` vs `Count` in the response:
- `ScannedCount: 4` — items read from the partition
- `Count: 2` — items returned after filtering

This is why filter expressions are not a substitute for a good key or index design.

---

## Task 4 — Projection expressions

A Query on `userId = "u-001"` with projection expression `userId, #ts, amount` returns only those three attributes per item. The `type` attribute is not included in the response.

`#ts` is an expression attribute name alias (because `timestamp` is a reserved word in DynamoDB).

The read cost is the same — projection only reduces network transfer, not RCU consumption.

---

## Task 5 — Scan with parallel segments

A table `large-catalog` exists with 1,000 items (partition key: `itemId`, String).

A Scan with no conditions returns all 1,000 items (paginated in chunks). The response includes a `LastEvaluatedKey` when there are more items to retrieve. The next Scan uses this key as `ExclusiveStartKey` to continue from where it left off.

A parallel Scan uses `TotalSegments: 4` and runs 4 simultaneous Scan operations (segments 0, 1, 2, 3). Each segment reads a quarter of the table independently. The combined results equal the full table — useful for bulk exports where speed matters more than cost.

---

## Task 6 — Limit

A Query on `userId = "u-001"` with `Limit: 2` returns at most 2 items (the first 2 by sort key). If `LastEvaluatedKey` is present in the response, more items exist and a subsequent Query with `ExclusiveStartKey` retrieves them.

**Important:** `Limit` applies to the number of items DynamoDB evaluates, not the number returned after filtering. A Query with `Limit: 2` and a filter expression may return 0 or 1 items if only some of the 2 evaluated items pass the filter.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Query requirement | Exact partition key value — always required |
| Sort key in Query | Optional; supports comparison and `begins_with` |
| Filter expression cost | Reads (and charges) all items before filtering |
| `ScannedCount` | Items read; `Count` = items returned |
| Scan cost | Reads every item — avoid on large tables |
| Pagination | Use `LastEvaluatedKey` / `ExclusiveStartKey` |
| Default sort order | Ascending by sort key |

---

## Clean up

Tables `events` and `large-catalog` are deleted.
