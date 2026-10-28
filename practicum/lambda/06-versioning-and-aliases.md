# Lab 06 — Versioning and Aliases

## Goals
- Immutable function versions exist alongside a mutable `$LATEST`
- Aliases point to specific versions and act as stable invocation targets
- A weighted alias splits traffic between two versions (canary deployment)
- Provisioned concurrency is configured on an alias

---

## How it works

```
Function: versioning-lab
  ├── $LATEST          ← mutable, always the most recent code
  ├── Version 1        ← immutable snapshot
  ├── Version 2        ← immutable snapshot
  └── Aliases
       ├── prod  → Version 1           (stable)
       ├── beta  → v1 (90%) + v2 (10%) (canary)
       └── dev   → $LATEST
```

Versions are immutable snapshots of code + configuration. Aliases are named pointers to a version — callers use the alias ARN and never need to update their configuration when you deploy.

---

## Task 1 — Publish versions

A function `versioning-lab` (Python 3.12) has two published versions:

**Version 1** (immutable):
```python
def lambda_handler(event, context):
    return {"version": "v1", "message": "Hello from version 1"}
```

**Version 2** (immutable):
```python
def lambda_handler(event, context):
    return {"version": "v2", "message": "Hello from version 2"}
```

`$LATEST` reflects Version 2's code. When Version 1 is selected from the qualifiers dropdown, invoking it returns `v1`. When Version 2 is selected, it returns `v2`. Neither version can be edited.

---

## Task 2 — Create aliases

Two aliases exist on `versioning-lab`:

| Alias | Points to |
|-------|-----------|
| `prod` | Version 1 |
| `dev` | `$LATEST` |

Invoking the `prod` alias returns `v1`. Invoking the `dev` alias returns `v2` (because `$LATEST` is the latest deployment).

Aliases have their own ARNs, e.g.:
```
arn:aws:lambda:eu-west-1:<account>:function:versioning-lab:prod
```

Callers use this ARN — when `prod` is later updated to point to Version 2, callers need no changes.

---

## Task 3 — Weighted traffic (canary deployment)

The `prod` alias is configured with weighted routing:

| Version | Weight |
|---------|--------|
| 1 | 90% |
| 2 | 10% |

After invoking the `prod` alias ~20 times, CloudWatch Logs shows both `v1` and `v2` log entries, roughly in a 9:1 ratio.

When confidence in Version 2 is high, `prod` is updated to route 100% to Version 2. The alias ARN remains unchanged — callers are unaffected.

---

## Task 4 — Provisioned concurrency

The `prod` alias has **provisioned concurrency set to 2**.

After invoking the alias several times, CloudWatch log entries for those first two concurrent executions contain no `Init Duration` line — the execution environments were pre-warmed. A third simultaneous invocation (beyond the provisioned slots) triggers a cold start and shows `Init Duration`.

> Provisioned concurrency can only be set on a **version** or **alias**, not on `$LATEST`.

---

## Key facts to remember

| Concept | Detail |
|---------|--------|
| `$LATEST` | Mutable — always reflects the latest deployment |
| Published version | Immutable — code + config frozen at publish time |
| Alias | A named pointer to a version or `$LATEST` |
| Weighted alias | Up to 2 versions; weights must sum to 100% |
| Provisioned concurrency | Set on a version or alias, not `$LATEST` |
| Version ARN format | `arn:aws:lambda:…:function:name:1` |
| Alias ARN format | `arn:aws:lambda:…:function:name:prod` |

---

## Clean up

The `prod` and `dev` aliases are deleted. `versioning-lab` is deleted (versions are deleted automatically with the function). Provisioned concurrency is removed before deletion to avoid charges.
