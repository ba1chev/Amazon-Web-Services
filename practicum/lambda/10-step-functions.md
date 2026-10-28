# Lab 10 — Orchestration with Step Functions

## Goals
- Multiple Lambda functions are coordinated by a Step Functions state machine
- Sequential and parallel execution patterns are understood
- Error handling and retries at the state machine level are configured
- The difference between orchestration and direct chaining is clear

---

## Why not just chain Lambda functions directly?

Calling Lambda A from Lambda B works but has problems:
- Both functions are running (and being billed) while waiting for the other
- Error handling and retries are spread across multiple functions
- The overall flow is invisible — you can't see where a workflow is stuck

Step Functions solves this by being the orchestrator. Each Lambda function does one job, Step Functions decides what runs next.

---

## Task 1 — Three single-purpose Lambda functions

Three functions exist (Python 3.12), each doing one step of an order processing workflow:

**`validate-order`** — checks that the order has required fields:
```python
def lambda_handler(event, context):
    if not event.get("item") or not event.get("quantity"):
        raise ValueError("Missing required fields: item, quantity")
    return {**event, "validated": True}
```

**`charge-payment`** — simulates a payment:
```python
import random
def lambda_handler(event, context):
    if random.random() < 0.2:   # 20% failure rate for testing
        raise RuntimeError("Payment gateway timeout")
    return {**event, "payment": "charged"}
```

**`ship-order`** — simulates shipping:
```python
def lambda_handler(event, context):
    return {**event, "status": "shipped", "tracking": "TRK-12345"}
```

Each function is tested independently before wiring into the state machine.

---

## Task 2 — Sequential state machine

A Step Functions Standard Workflow named `order-pipeline` exists. Its definition:

```json
{
  "Comment": "Order processing pipeline",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:<region>:<account>:function:validate-order",
      "Next": "ChargePayment"
    },
    "ChargePayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:<region>:<account>:function:charge-payment",
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:<region>:<account>:function:ship-order",
      "End": true
    }
  }
}
```

The IAM role attached to the state machine has `lambda:InvokeFunction` on all three function ARNs.

After starting an execution with `{"item": "book", "quantity": 2}`, the Step Functions console shows the execution graph with each state turning green in sequence. The final output contains `"status": "shipped"`.

---

## Task 3 — Retry on failure

`charge-payment` fails ~20% of the time. Without retry configuration, the state machine fails when this happens.

The `ChargePayment` state is updated with a retry policy:

```json
"ChargePayment": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:<region>:<account>:function:charge-payment",
  "Retry": [{
    "ErrorEquals": ["RuntimeError"],
    "IntervalSeconds": 2,
    "MaxAttempts": 3,
    "BackoffRate": 2
  }],
  "Next": "ShipOrder"
}
```

After re-running executions, the Step Functions console shows retry attempts as loops on the `ChargePayment` node in the execution graph. Executions that previously failed now complete successfully after one or two retries.

---

## Task 4 — Catch and handle errors

The state machine catches validation failures and routes them to a dedicated failure state instead of aborting:

```json
"ValidateOrder": {
  "Type": "Task",
  "Resource": "arn:aws:lambda:<region>:<account>:function:validate-order",
  "Catch": [{
    "ErrorEquals": ["ValueError"],
    "Next": "OrderRejected"
  }],
  "Next": "ChargePayment"
},
"OrderRejected": {
  "Type": "Fail",
  "Error": "ValidationError",
  "Cause": "Order failed validation"
}
```

When started with `{"quantity": 2}` (missing `item`), the execution transitions to `OrderRejected` instead of an unhandled failure. The console shows the path taken through the graph.

---

## Task 5 — Parallel execution

A new state machine named `parallel-pipeline` runs two independent checks simultaneously before proceeding:

```json
{
  "StartAt": "ParallelChecks",
  "States": {
    "ParallelChecks": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "ValidateOrder",
          "States": {
            "ValidateOrder": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:<region>:<account>:function:validate-order",
              "End": true
            }
          }
        },
        {
          "StartAt": "CheckInventory",
          "States": {
            "CheckInventory": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:<region>:<account>:function:validate-order",
              "End": true
            }
          }
        }
      ],
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:<region>:<account>:function:ship-order",
      "End": true
    }
  }
}
```

Both branches run simultaneously. `ShipOrder` only starts after **both** branches complete. The total execution time is the duration of the slower branch, not the sum of both.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Standard vs Express | Standard: up to 1 year, at-most-once, auditable; Express: up to 5 min, at-least-once, high throughput |
| Retry in state machine | Defined per state — keeps retry logic out of function code |
| Parallel state | All branches run simultaneously; output is an array of each branch's result |
| Step Functions billing | Standard: per state transition; Express: per execution + duration |
| Lambda billing | Only billed for actual execution time — no charge while Step Functions waits |

---

## Clean up

Functions `validate-order`, `charge-payment`, and `ship-order` are deleted.
State machines `order-pipeline` and `parallel-pipeline` are deleted.
The IAM role created for the state machine is deleted.
