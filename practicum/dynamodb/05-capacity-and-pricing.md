# Lab 05 — Capacity and Pricing

## Goals
- RCUs and WCUs are calculated from actual item sizes
- Provisioned capacity is set, monitored, and auto-scaled
- On-demand capacity requires no planning and scales automatically
- Throttling behavior is observed and understood

---

## Capacity unit rules

| Operation | Cost |
|-----------|------|
| 1 WCU | 1 write of up to 1 KB |
| 1 RCU | 1 strongly consistent read of up to 4 KB |
| 0.5 RCU | 1 eventually consistent read of up to 4 KB |
| 2 WCU | 1 transactional write of up to 1 KB |
| 2 RCU | 1 transactional read of up to 4 KB |

Items are rounded up to the nearest 1 KB (writes) or 4 KB (reads).

**Example:** A 3.5 KB item costs 4 WCUs to write (rounded up to 4 KB × 1 WCU/KB), and 1 RCU to read (3.5 KB ≤ 4 KB).

---

## Task 1 — Calculate RCU and WCU manually

A table `metrics` exists (partition key: `metricId`, String, on-demand billing).

The following items are written. Before writing each one, calculate the expected WCU cost:

```
Item A: 800 bytes   → expected WCU: ?
Item B: 1,100 bytes → expected WCU: ?
Item C: 4,500 bytes → expected WCU: ?
Item D: 9,000 bytes → expected WCU: ?
```

After writing, check the `ConsumedCapacity` field in the response (enable **Return consumed capacity: TOTAL** in the API call or console) to confirm the actual cost matches the calculation.

Answers:
- Item A: 1 WCU (800 B rounds up to 1 KB)
- Item B: 2 WCU (1,100 B rounds up to 2 KB)
- Item C: 5 WCU (4,500 B rounds up to 5 KB)
- Item D: 9 WCU (9,000 B rounds up to 9 KB)

---

## Task 2 — Provisioned capacity and throttling

A table `throttle-test` exists with:
- Partition key: `id` (String)
- Billing mode: **Provisioned**
- Read capacity: **5 RCU**
- Write capacity: **2 WCU**

Items are written at a rate that exceeds 2 WCU/second (e.g., 10 small writes in rapid succession from the console or a script).

The CloudWatch metric `WriteThrottleEvents` increases. Some write requests are rejected with `ProvisionedThroughputExceededException`.

The AWS SDK's built-in retry logic (with exponential backoff) automatically retries throttled requests — most callers never see the error unless the rate is sustained for a long time.

After raising write capacity to **20 WCU**, the same burst of writes completes without throttling.

---

## Task 3 — On-demand vs provisioned

A second table `on-demand-test` exists with identical structure but **On-demand** billing.

The same burst of writes succeeds immediately with no throttling — DynamoDB scales capacity automatically.

**Trade-off:** On-demand costs more per request than provisioned at steady, predictable loads. Provisioned is cheaper when you can forecast traffic accurately. On-demand is better for unpredictable or spiky workloads.

---

## Task 4 — Auto-scaling on provisioned capacity

The `throttle-test` table has Application Auto Scaling configured:
- Minimum write capacity: 2 WCU
- Maximum write capacity: 50 WCU
- Target utilization: 70%

A sustained write load is applied. After a few minutes, the CloudWatch metric `ProvisionedWriteCapacityUnits` increases automatically as DynamoDB scales up to meet demand.

**Observation:** Auto-scaling reacts over minutes, not seconds. A sudden burst can still cause throttling before scaling catches up. Provisioned concurrency or on-demand is better for immediate bursts.

---

## Task 5 — Hot partitions

A table `hot-partition-test` exists (partition key: `type`, String, 10 WCU provisioned).

All writes use `type = "product"` as the partition key. Even though the table has 10 WCU, all traffic goes to a single partition which has a per-partition limit.

Throttling occurs even though the total table capacity appears sufficient.

A redesigned table uses a **high-cardinality** partition key (e.g., `productId` with many unique values) so writes are distributed across partitions. The throttling disappears.

**Key insight:** DynamoDB distributes capacity evenly across partitions. A low-cardinality partition key concentrates all traffic on one partition, creating a hot partition that throttles before the table limit is reached.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| WCU rounding | Up to nearest 1 KB |
| RCU rounding | Up to nearest 4 KB |
| Throttle exception | `ProvisionedThroughputExceededException` |
| SDK retry | Built-in exponential backoff handles transient throttling |
| Auto-scaling lag | Minutes — not suitable for instant bursts |
| Hot partition | Caused by low-cardinality or sequential partition keys |
| On-demand pricing | Per request — no idle capacity cost |

---

## Clean up

Tables `metrics`, `throttle-test`, `on-demand-test`, and `hot-partition-test` are deleted.
