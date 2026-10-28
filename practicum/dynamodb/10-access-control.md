# Lab 10 — Access Control

## Goals
- IAM policies control which tables and operations a principal can access
- Fine-grained access control (FGAC) restricts access to specific items using IAM conditions
- A Lambda function's execution role follows least privilege for DynamoDB
- Resource-based policies are not used with DynamoDB — IAM is the only mechanism

---

## How DynamoDB access control works

DynamoDB has no resource-based policies. All access is controlled through **IAM identity-based policies** attached to users, roles, or groups. The principal making the request must have the required `dynamodb:*` action allowed on the table ARN.

---

## Task 1 — Table-level IAM policy

A table `orders` exists (partition key: `customerId`, sort key: `orderId`).

An IAM role `orders-read-role` has the following inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query",
        "dynamodb:BatchGetItem"
      ],
      "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/orders"
    }
  ]
}
```

A Lambda function with this execution role can read from `orders` but:
- `PutItem` → denied
- `UpdateItem` → denied
- `DeleteItem` → denied
- `Scan` → denied (not in the allow list)

**Observation:** Listing individual actions rather than `dynamodb:*` enforces least privilege. A scan is explicitly excluded — a Scan on a large table would consume significant RCUs and should be controlled.

---

## Task 2 — Index access

A GSI `status-index` exists on `orders`. A Query against this index fails even though the policy allows `dynamodb:Query` on the table ARN.

The policy must also allow the index ARN:

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:Query"],
  "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/orders/index/status-index"
}
```

After adding this statement, Queries on `status-index` succeed. The base table Query still works with the original table ARN statement.

---

## Task 3 — Fine-grained access control (item-level)

A table `user-data` exists (partition key: `userId`, String). Each item belongs to one user.

An IAM policy for a mobile app's execution role restricts reads to only the items belonging to the authenticated user. The policy uses the `dynamodb:LeadingKeys` condition key:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/user-data",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${cognito-identity.amazonaws.com:sub}"]
        }
      }
    }
  ]
}
```

The `${cognito-identity.amazonaws.com:sub}` placeholder is resolved at request time to the authenticated user's Cognito identity ID. A user can only access items where `userId` equals their own identity.

**Verification:** A request with `userId = "u-001"` reading `userId = "u-002"` data is rejected with `AccessDeniedException` — even though the role has `GetItem` allowed on the table.

---

## Task 4 — Restrict writable attributes

The `dynamodb:Attributes` condition key restricts which attributes a principal can read or write:

```json
{
  "Effect": "Allow",
  "Action": ["dynamodb:GetItem"],
  "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/orders",
  "Condition": {
    "ForAllValues:StringEquals": {
      "dynamodb:Attributes": ["orderId", "customerId", "status", "total"]
    },
    "StringEqualsIfExists": {
      "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
    }
  }
}
```

A GetItem requesting only `orderId` and `status` succeeds. A GetItem requesting `internalNotes` (not in the allowed list) is denied.

---

## Task 5 — Deny specific operations

A policy grants full DynamoDB access to a table except for `DeleteItem` and `Scan`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:*",
      "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/orders"
    },
    {
      "Effect": "Deny",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:<region>:<account-id>:table/orders"
    }
  ]
}
```

`DeleteItem` and `Scan` are denied even though `dynamodb:*` was allowed — **Deny always overrides Allow** in IAM evaluation.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Resource-based policies | DynamoDB does not support them — IAM only |
| GSI access | Separate `Resource` entry for the index ARN |
| `dynamodb:LeadingKeys` | Restricts access to items whose partition key matches the condition value |
| `dynamodb:Attributes` | Restricts which attributes can be accessed |
| Deny overrides Allow | An explicit Deny in any policy always wins |
| Cognito integration | Identity pool roles use `${cognito-identity.amazonaws.com:sub}` in conditions |

---

## Clean up

Tables `orders`, `user-data` are deleted.
IAM roles `orders-read-role` and any inline policies created during the lab are deleted.
