# Lab 04 — Environment Variables and Configuration

## Goals
- A function reads configuration from environment variables without any code changes between environments
- Sensitive values are encrypted with a KMS customer-managed key
- Secrets are fetched at runtime from SSM Parameter Store and Secrets Manager
- The initialization phase vs the handler phase is clear from the code structure

---

## Task 1 — Basic environment variables

A Lambda function `config-lab` (Python 3.12) reads three environment variables and returns them:

```python
import os, json

def lambda_handler(event, context):
    return {
        "env":     os.environ.get("APP_ENV", "not set"),
        "db_host": os.environ.get("DB_HOST", "not set"),
        "debug":   os.environ.get("DEBUG", "false"),
    }
```

The function's configuration has these environment variables set:

| Key | Value |
|-----|-------|
| `APP_ENV` | `production` |
| `DB_HOST` | `mydb.cluster.local` |
| `DEBUG` | `false` |

When invoked with `{}`, the response contains all three values.

After changing `APP_ENV` to `staging` in the configuration (no code redeploy), the next invocation returns `staging`. This demonstrates that environment variables can change behavior without touching code.

---

## Task 2 — KMS encryption

The function's environment variables are encrypted using a **customer-managed KMS key** (CMK) named `lambda-lab-key`.

The execution role has an inline policy granting `kms:Decrypt` on that key:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "kms:Decrypt",
    "Resource": "arn:aws:kms:<region>:<account-id>:key/<key-id>"
  }]
}
```

A sensitive variable `SECRET_TOKEN` is stored encrypted. When invoked, Lambda decrypts it transparently and the function reads it via `os.environ`. The plaintext never appears in the console.

---

## Task 3 — SSM Parameter Store

A SecureString parameter `/myapp/db/password` exists in Systems Manager Parameter Store with value `s3cr3tP@ss`.

The execution role has `ssm:GetParameter` on that parameter's ARN, and `kms:Decrypt` on the key used to encrypt it.

The function fetches the parameter **outside the handler** so it is loaded once during initialization and reused on warm invocations:

```python
import boto3

ssm = boto3.client("ssm")

# Runs once per execution environment (initialization phase)
DB_PASSWORD = ssm.get_parameter(
    Name="/myapp/db/password",
    WithDecryption=True
)["Parameter"]["Value"]

def lambda_handler(event, context):
    return {"password_length": len(DB_PASSWORD)}
```

When invoked multiple times without a cold start, the SSM API is not called again — the cached value is reused.

---

## Task 4 — Secrets Manager

A secret `myapp/db-creds` exists in Secrets Manager:

```json
{"username": "admin", "password": "s3cr3tP@ss"}
```

The execution role has `secretsmanager:GetSecretValue` on that secret's ARN.

The function retrieves and parses the secret at initialization:

```python
import boto3, json

sm = boto3.client("secretsmanager")

SECRET = json.loads(
    sm.get_secret_value(SecretId="myapp/db-creds")["SecretString"]
)

def lambda_handler(event, context):
    return {"username": SECRET["username"]}
```

When invoked, the response contains `admin` without any credentials hardcoded in the function code.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| Max env vars payload | 4 KB total |
| Default encryption | AWS-managed key (automatic, free) |
| CMK encryption | Execution role needs `kms:Decrypt` on the key |
| SSM SecureString | Execution role needs `ssm:GetParameter` + `kms:Decrypt` |
| Caching pattern | Initialize SDK calls outside the handler — reused on warm starts |
| Reserved env vars | `AWS_REGION`, `AWS_LAMBDA_FUNCTION_NAME` — read-only, cannot be overridden |

---

## Clean up

`config-lab` is deleted. The SSM parameter, Secrets Manager secret, and KMS key are also deleted (KMS key deletion has a mandatory 7-day waiting period).
