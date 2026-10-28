# Lab 09 — Lambda Inside a VPC

## Goals
- A VPC-attached function can reach private resources (RDS, ElastiCache)
- A VPC-attached function without a NAT Gateway cannot reach the internet
- Internet access is restored when a NAT Gateway exists in a public subnet
- Security group rules control which ports are reachable from Lambda

---

## Why put Lambda in a VPC?

By default Lambda runs in an AWS-managed network — it can reach the internet but not private VPC resources.
Attaching Lambda to a VPC subnet allows it to reach private resources, but removes internet access unless a NAT Gateway is in the path.

```
Without VPC:   Lambda ──► Internet ✓   Private VPC resources ✗
With VPC:      Lambda ──► Internet ✗   Private VPC resources ✓
With VPC+NAT:  Lambda ──► Internet ✓   Private VPC resources ✓
```

---

## Task 1 — VPC prerequisites

A VPC named `lambda-lab-vpc` exists with:
- 2 public subnets (one per AZ)
- 2 private subnets (one per AZ)
- An Internet Gateway attached to the VPC
- A NAT Gateway in one of the public subnets
- A route in the **private** route table: `0.0.0.0/0 → nat-...`
- A security group named `lambda-sg` with no inbound rules and outbound `All traffic` allowed

---

## Task 2 — VPC-attached function without internet

A function `vpc-lab` (Python 3.12) is deployed into the **private** subnets of `lambda-lab-vpc` using `lambda-sg`.

The execution role has `AWSLambdaVPCAccessExecutionRole` attached (required for Lambda to create network interfaces in the VPC).

The handler tries to reach a public URL:

```python
import urllib.request

def lambda_handler(event, context):
    try:
        with urllib.request.urlopen("https://httpbin.org/get", timeout=5) as r:
            return {"status": r.status}
    except Exception as e:
        return {"error": str(e)}
```

When invoked, the request times out — the private subnets have no direct route to the internet.

EC2 → Network interfaces shows Elastic Network Interfaces (ENIs) created by Lambda in the private subnets. Lambda reuses these ENIs across warm invocations.

---

## Task 3 — Restore internet access via NAT Gateway

The private route table already has a route `0.0.0.0/0 → nat-...` (from Task 1).

After verifying this route is in place and invoking `vpc-lab` again, the request to `httpbin.org` succeeds and returns HTTP 200.

The traffic path: Lambda ENI → private subnet → NAT Gateway → Internet Gateway → internet.

---

## Task 4 — Private RDS access

An RDS MySQL instance (Free Tier) named `lab-db` exists in `lambda-lab-vpc` with:
- Public access: **disabled**
- VPC security group: `rds-sg`

`rds-sg` has an inbound rule: **MySQL/Aurora (port 3306)** from `lambda-sg`.

This means only Lambda (via `lambda-sg`) can reach the database — not the public internet.

The function connects to RDS using `pymysql` (provided via a Layer):

```python
import pymysql, os

connection = pymysql.connect(
    host=os.environ["DB_HOST"],
    user=os.environ["DB_USER"],
    password=os.environ["DB_PASS"],
    database="mysql",
    connect_timeout=5
)

def lambda_handler(event, context):
    with connection.cursor() as cursor:
        cursor.execute("SELECT VERSION()")
        version = cursor.fetchone()
    return {"mysql_version": version[0]}
```

Environment variables `DB_HOST`, `DB_USER`, and `DB_PASS` are set in the function configuration (or fetched from Secrets Manager as in Lab 04).

When invoked, the response contains the MySQL version string.

When `lambda-sg` is removed from `rds-sg`'s inbound rules, the connection times out — demonstrating that security group rules are the access control mechanism between VPC resources.

---

## Key facts to remember

| Topic | Detail |
|-------|--------|
| ENI creation | Lambda creates ENIs in selected subnets; reused across warm invocations |
| Internet access | Requires NAT Gateway in a public subnet + default route in private route table |
| Required execution role | `AWSLambdaVPCAccessExecutionRole` |
| Security group (Lambda) | No inbound rules needed; outbound must allow the target port |
| Security group (RDS) | Inbound rule references Lambda's security group by ID, not by IP |
| Multiple subnets | Subnets in multiple AZs for high availability |

---

## Clean up

`vpc-lab` is deleted. The RDS instance is deleted. The NAT Gateway is deleted and its Elastic IP released (both incur hourly charges even when idle). `lambda-lab-vpc` and its subnets can be kept for other labs or deleted.
