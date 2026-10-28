# AWS DynamoDB — Practical Labs

Hands-on labs for understanding DynamoDB in depth.
Each lab is self-contained and runs within the AWS Free Tier.

## Labs

| File | Topic |
|------|-------|
| [01-tables-and-items.md](01-tables-and-items.md) | Tables, partition keys, sort keys, items |
| [02-read-write-operations.md](02-read-write-operations.md) | PutItem, GetItem, UpdateItem, DeleteItem |
| [03-queries-and-scans.md](03-queries-and-scans.md) | Query vs Scan, filter expressions, projection |
| [04-indexes.md](04-indexes.md) | GSI and LSI — when and why |
| [05-capacity-and-pricing.md](05-capacity-and-pricing.md) | Provisioned vs on-demand, RCUs, WCUs |
| [06-streams.md](06-streams.md) | DynamoDB Streams and change capture |
| [07-transactions.md](07-transactions.md) | TransactWriteItems, TransactGetItems |
| [08-ttl.md](08-ttl.md) | Time To Live — automatic item expiry |
| [09-conditional-writes.md](09-conditional-writes.md) | Conditional expressions and optimistic locking |
| [10-access-control.md](10-access-control.md) | IAM policies, fine-grained access control |

## Before you start

All labs use the **eu-west-1** or **us-east-1** region.
Items shown in code use the DynamoDB JSON format (e.g. `{"S": "value"}` for strings).
