# Lab 03 — IAM Roles and Permissions

## Goals
- Understand the difference between an execution role and a resource-based policy
- Grant a function access to another AWS service using a least-privilege policy
- Understand what happens when a required permission is missing

---

## Concept recap

| Concept | What it controls |
|---------|-----------------|
| **Execution role** (identity-based policy) | What Lambda *can do* — e.g., write to S3, read from DynamoDB |
| **Resource-based policy** | Who *can invoke* the Lambda function — e.g., API Gateway, S3, another account |

Both must be correct. A missing execution role permission causes a runtime error inside the function. A missing resource-based policy prevents the trigger from invoking the function at all.

---

## Task 1 — Inspect the default execution role

A Lambda function `iam-lab` exists with default settings (Python 3.12). Its auto-created execution role has exactly one policy attached: `AWSLambdaBasicExecutionRole`.

The policy allows only:
- `logs:CreateLogGroup`
- `logs:CreateLogStream`
- `logs:PutLogEvents`

This is the minimum needed to write to CloudWatch Logs. Nothing else is permitted.

---

## Task 2 — Access S3 without permission (expected failure)

The function attempts to read an object from S3:

```python
import boto3

s3 = boto3.client("s3")
BUCKET = "your-existing-bucket"

def lambda_handler(event, context):
    try:
        obj = s3.get_object(Bucket=BUCKET, Key="test.txt")
        return {"content": obj["Body"].read().decode()}
    except Exception as e:
        return {"error": str(e)}
```

When invoked, the response contains `AccessDenied`. This confirms the execution role controls what the function can do.

---

## Task 3 — Grant least-privilege S3 access

The execution role has an inline policy granting only `s3:GetObject` on a specific bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::your-existing-bucket/*"
    }
  ]
}
```

After attaching this policy, `s3.get_object` succeeds.

An attempt to call `s3.put_object` still fails with `AccessDenied` — the policy grants only `GetObject`, not `PutObject`. Least privilege is enforced.

---

## Task 4 — Resource-based policy

A trigger attached to a function (S3, API Gateway, SNS) only works when the function's **resource-based policy** grants the triggering service `lambda:InvokeFunction`.

The function's resource-based policy (visible under **Configuration → Permissions → Resource-based policy statements**) contains a statement like:

```json
{
  "Effect": "Allow",
  "Principal": {"Service": "s3.amazonaws.com"},
  "Action": "lambda:InvokeFunction",
  "Condition": {
    "ArnLike": {"AWS:SourceArn": "arn:aws:s3:::your-bucket"}
  }
}
```

When this statement is missing, S3 cannot invoke the function even if the trigger is configured.

To observe this: remove the resource-based policy statement and attempt to trigger the function from S3. The invocation fails silently from the S3 side (S3 logs an error; Lambda is never called).

---

## Task 5 — Cross-account invocation

The function's resource-based policy has a statement granting another AWS account `lambda:InvokeFunction`:

```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::<OTHER_ACCOUNT_ID>:root"},
  "Action": "lambda:InvokeFunction"
}
```

A principal in the other account (with the appropriate IAM permission on their side) can invoke the function.

---

## Key facts to remember

- Lambda always needs `AWSLambdaBasicExecutionRole` to write logs — without it the function runs but nothing appears in CloudWatch.
- The console adds resource-based policy statements automatically when configuring triggers. The CLI requires an explicit `add-permission` call.
- A resource-based policy can be at most **20 KB**.
- VPC-attached functions also need `AWSLambdaVPCAccessExecutionRole` in the execution role.

---

## Clean up

`iam-lab` is deleted. The inline policy on the execution role is removed if the role will be reused.
