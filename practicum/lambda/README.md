# AWS Lambda — Practical Labs

Hands-on labs for understanding AWS Lambda in depth.
Each lab is self-contained and runs within the AWS Free Tier.

## Labs

| File | Topic |
|------|-------|
| [01-first-function.md](01-first-function.md) | First function, configuration, testing |
| [02-triggers-and-events.md](02-triggers-and-events.md) | Triggers: API Gateway, S3, DynamoDB Streams |
| [03-iam-and-permissions.md](03-iam-and-permissions.md) | IAM roles, execution role, resource-based policy |
| [04-environment-and-config.md](04-environment-and-config.md) | Environment variables, SSM Parameter Store |
| [05-layers.md](05-layers.md) | Lambda Layers — shared dependencies |
| [06-versioning-and-aliases.md](06-versioning-and-aliases.md) | Versions, aliases, traffic shifting |
| [07-concurrency.md](07-concurrency.md) | Reserved & provisioned concurrency, throttling |
| [08-error-handling.md](08-error-handling.md) | Retries, DLQ, destinations |
| [09-vpc.md](09-vpc.md) | Lambda inside a VPC, ENI, RDS access |
| [10-step-functions.md](10-step-functions.md) | Orchestration with Step Functions |

## Prerequisites

```bash
aws --version                  # AWS CLI v2
aws sts get-caller-identity    # verify credentials
```

Use **eu-west-1** or **us-east-1** for all labs.
