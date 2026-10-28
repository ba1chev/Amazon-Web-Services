# Lab 07 — Transactions

## Goals
- `TransactWriteItems` applies multiple writes atomically across items and tables
- `TransactGetItems` reads multiple items in a consistent snapshot
- The cost difference between transactional and non-transactional operations is understood
- Transactions are used only when atomicity is genuinely required

---

## Why transactions?

`BatchWriteItem` can write multiple items in one request but is **not atomic** — some items may succeed while others fail. Transactions guarantee all-or-nothing: either every operation succeeds, or none of them do.

Transactions are useful for:
- Transferring a balance between two accounts (debit one, credit another)
- Placing an order (decrement stock, create order record)
- Enforcing a uniqueness constraint on a non-key attribute

---

## Task 1 — Atomic balance transfer

Two tables exist:

**`accounts`** (partition key: `accountId`, String):
```
accountId: "acc-001"   owner: "Alice"   balance: 500
accountId: "acc-002"   owner: "Bob"     balance: 100
```

A transfer of 150 from Alice to Bob is performed as a single `TransactWriteItems` call containing two `Update` operations:

```
Operation 1: Update acc-001, SET balance = balance - :amount
             Condition: balance >= :amount   (prevent overdraft)

Operation 2: Update acc-002, SET balance = balance + :amount
```

After the transaction, Alice has 350 and Bob has 250.

**Failure scenario:** A second transfer of 400 from Alice (current balance: 350) is attempted. The condition on Operation 1 fails (`350 < 400`). The entire transaction is rejected — Bob's balance remains unchanged. This is the all-or-nothing guarantee.

---

## Task 2 — Uniqueness constraint via conditional transaction

A table `usernames` (partition key: `username`, String) enforces unique usernames even though uniqueness is usually handled by the key itself.

A table `user-profiles` (partition key: `userId`, String) stores user data.

Registering a new user atomically:

```
TransactWriteItems:
  Operation 1: Put into usernames — item {username: "alice"}
               Condition: attribute_not_exists(username)   ← fail if already taken

  Operation 2: Put into user-profiles — item {userId: "u-001", username: "alice", email: "..."}
```

If `alice` is already in `usernames`, both operations are rejected. The profile is never created without the username reservation succeeding.

A second attempt to register `alice` fails with `TransactionCanceledException`. The `CancellationReasons` field identifies which operation caused the failure and why (`ConditionalCheckFailed`).

---

## Task 3 — TransactGetItems

A consistent read of multiple items across tables in a single snapshot:

```
TransactGetItems:
  Get from accounts: accountId = "acc-001"
  Get from accounts: accountId = "acc-002"
  Get from user-profiles: userId = "u-001"
```

All three items are returned at the same point in time — no race condition between reads. If a concurrent write modifies one of the items mid-read, the transaction is retried automatically.

**Cost:** Each item in a `TransactGetItems` costs **2 RCU** (compared to 1 RCU for a standard strongly consistent read).

---

## Task 4 — Transaction idempotency

A `TransactWriteItems` call includes an optional `ClientRequestToken` (a unique string you generate, e.g., a UUID).

If the same token is submitted again within **10 minutes**, DynamoDB returns the same result without re-applying the writes. This prevents duplicate charges if a network timeout causes the client to retry a successful transaction.

Two transactions with identical `ClientRequestToken` but different operations are submitted. The second call returns the original result — the second set of operations is ignored.

---

## Task 5 — Failure modes

**`TransactionConflictException`:** Two concurrent transactions attempt to modify the same item at the same time. One succeeds; the other is rejected. The caller must retry.

**`TransactionCanceledException`:** One or more operations in the transaction failed (e.g., a condition check). The `CancellationReasons` array identifies each operation's outcome (`None` for success, `ConditionalCheckFailed`, `ItemCollectionSizeLimitExceeded`, etc.).

**`IdempotentParameterMismatch`:** A `ClientRequestToken` is reused with a different set of operations within the 10-minute deduplication window.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Max items per TransactWrite | 100 operations across ≤ 100 items |
| Max items per TransactGet | 100 items |
| Transaction cost | 2× the standard WCU/RCU cost |
| Atomicity | All succeed or all fail — no partial writes |
| Idempotency window | 10 minutes via `ClientRequestToken` |
| Cross-table | Transactions can span multiple tables in the same region |
| Cross-region | Not supported — transactions are region-scoped |

---

## Clean up

Tables `accounts`, `usernames`, and `user-profiles` are deleted.
