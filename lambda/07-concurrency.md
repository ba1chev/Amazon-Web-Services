# Lab 07 — Concurrency

## Goals
- Lambda's automatic scaling behavior is observable in CloudWatch metrics
- Reserved concurrency limits a function and guarantees capacity
- Setting reserved concurrency to 0 disables the function
- Provisioned concurrency eliminates cold starts for the pre-warmed slots
- Throttling behavior and the 429 response are understood

---

## How Lambda concurrency works

Each simultaneous in-flight request runs in its own execution environment.
Lambda scales out by launching new environments in parallel — up to the account limit.

```
Request 1 ──► Env A (warm)
Request 2 ──► Env B (warm)
Request 3 ──► Env C (cold start)
Request 4 ──► 429 TooManyRequestsException  (if reserved limit is reached)
```

**Account-level default:** 1,000 concurrent executions per region (soft limit).

---

## Concurrency types

| Type | What it does |
|------|-------------|
| **Unreserved** | Drawn from the shared account pool |
| **Reserved** | Guaranteed for one function; also acts as a hard cap |
| **Provisioned** | Pre-warms N environments — no cold start for those slots |

---

## Task 1 — Observe scaling in CloudWatch

A function `concurrency-lab` (Python 3.12) has a 2-second artificial delay:

```python
import time

def lambda_handler(event, context):
    time.sleep(2)
    return {"done": True}
```

After sending ~20 simultaneous invocations, the CloudWatch metrics for the function show:

- **ConcurrentExecutions** — the peak number of simultaneous executions
- **Throttles** — any invocations that were rejected
- **Duration (p99)** — tail latency

Lambda scaled out automatically without any configuration change.

---

## Task 2 — Reserved concurrency as a cap

The function's reserved concurrency is set to **5**.

After firing more than 5 overlapping requests (each lasting 2 s), the responses beyond the 5th contain:

```json
{"errorMessage": "Rate Exceeded.", "errorType": "TooManyRequestsException"}
```

The **Throttles** metric in CloudWatch increases.

The remaining account concurrency pool is unaffected — the 5 reserved slots are guaranteed to this function and cannot be consumed by others, but this function cannot exceed them either.

---

## Task 3 — Reserved concurrency = 0 (disable the function)

Reserved concurrency is set to **0**.

Every invocation immediately returns `TooManyRequestsException` (HTTP 429). The function code never runs.

This is how a function is disabled without deleting it — useful to stop a runaway function quickly.

After confirming the behavior, the reserved concurrency is set back to a non-zero value (or removed to use the shared pool).

---

## Task 4 — Provisioned concurrency

A version (e.g., Version 1) of `concurrency-lab` is published. Provisioned concurrency is set to **2** on that version.

After provisioning completes, 5 rapid invocations of that version produce CloudWatch log entries where:
- The first 2 concurrent executions have **no `Init Duration`** (warm environments)
- The 3rd+ concurrent invocation shows `Init Duration` (overflows into on-demand capacity)

> Provisioned concurrency requires a published version or alias — it cannot be set on `$LATEST`.
> It incurs cost even when the function is idle.

---

## Task 5 — Account concurrency limit

The Lambda console's function list page shows a **Total concurrent executions** banner with the current usage versus the account limit.

The Service Quotas console (under Lambda) shows the adjustable quota for **Concurrent executions**. This is where a limit increase request would be submitted.

---

## Key facts to remember

| Topic | Value / Behaviour |
|-------|------------------|
| Default account limit | 1,000 concurrent executions per region |
| Throttle response | HTTP 429 / `TooManyRequestsException` |
| Reserved concurrency = 0 | Function is disabled |
| Provisioned concurrency target | A version or alias — not `$LATEST` |
| Burst limit | 500–3,000 new environments per minute (region-dependent) |
| Cold start with provisioned | Only happens when provisioned slots are all occupied |

---

## Clean up

Provisioned concurrency configuration is removed (to stop charges). `concurrency-lab` is deleted.
