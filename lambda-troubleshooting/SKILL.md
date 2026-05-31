---
name: lambda-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Lambda function
  problems by analyzing CloudWatch logs, metrics, configurations, and
  following structured runbooks. Activate when: a function is timing
  out, returning errors (5xx, unhandled exceptions), cold starts are
  too slow, concurrency is throttled, functions can't reach VPC
  resources or the internet, memory/CPU is insufficient, deployment
  fails, layers are missing, permissions are denied (IAM, resource
  policy), event source mappings fail, destinations aren't triggered,
  or the user says something is wrong with a Lambda function without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Lambda, CloudWatch Logs,
  CloudWatch Metrics, IAM, and optionally VPC/X-Ray permissions.
---

# Lambda Diagnostics

## When to use

Any Lambda investigation where CloudWatch metrics alone are insufficient — function logs, configuration review, IAM policy analysis, VPC networking, concurrency limits, event source mapping state, layer compatibility, or deployment issues.

## Investigation workflow

### Step 1 — Collect and triage

```
# Get function configuration
aws lambda get-function-configuration --function-name <name>

# Get recent invocation errors
aws logs filter-log-events --log-group-name /aws/lambda/<name> --filter-pattern "ERROR"

# Get key metrics
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<name>
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Duration --dimensions Name=FunctionName,Value=<name>
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles --dimensions Name=FunctionName,Value=<name>
```

### Step 2 — Domain deep dive (only if needed)

```
# Concurrency
aws lambda get-function-concurrency --function-name <name>
aws lambda get-account-settings

# VPC networking
aws lambda get-function-configuration --function-name <name>  # VpcConfig
aws ec2 describe-subnets --subnet-ids <ids>
aws ec2 describe-security-groups --group-ids <ids>

# Event source mappings
aws lambda list-event-source-mappings --function-name <name>

# IAM
aws lambda get-policy --function-name <name>
aws iam get-role --role-name <role>
```

Read `references/lambda-guardrails.md` before concluding on any Lambda issue.

### Step 3 — Detailed path (low-confidence cases only)

```
# X-Ray traces (if enabled)
aws xray get-trace-summaries --start-time <time> --end-time <time> --filter-expression "service(\"<name>\")"

# CloudTrail for API-level events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<name>
```

## Gotchas: Lambda

- Timeout max is 900 seconds (15 minutes). If your function needs longer, Lambda is the wrong service.
- Memory setting controls BOTH memory AND proportional CPU. 1769 MB = 1 full vCPU. Below that, CPU is fractional.
- /tmp is the ONLY writable filesystem. Max 10 GB (configurable). Persists across warm invocations but NOT guaranteed.
- Cold starts are unavoidable for the first invocation. VPC cold starts improved dramatically with Hyperplane ENIs but still add latency.
- Provisioned Concurrency eliminates cold starts but you pay for it even when idle.
- Reserved Concurrency caps the function's max concurrency AND removes those units from the unreserved pool. Setting it too low throttles the function. Setting it too high starves other functions.
- Environment variables have a 4 KB total size limit. Use SSM Parameter Store or Secrets Manager for larger configs.
- Deployment package max: 50 MB zipped (direct upload), 250 MB unzipped, 10 GB container image.
- Layers are extracted to /opt. Total unzipped size of function + all layers must be ≤ 250 MB.
- VPC Lambda functions need NAT gateway for internet access. They do NOT get public IPs.
- Synchronous invocations return errors to the caller. Asynchronous invocations retry twice then go to DLQ/destination.
- Event source mappings (SQS, Kinesis, DynamoDB) have their own retry and error handling semantics.

### Concurrency model

| Setting | Effect |
|---------|--------|
| Unreserved account concurrency | Shared pool (default 1000 per region) |
| Reserved concurrency | Dedicated cap for this function, removed from shared pool |
| Provisioned concurrency | Pre-initialized execution environments (no cold starts) |

### Retry behavior

| Invocation type | Retry | Error handling |
|-----------------|-------|----------------|
| Synchronous | No retry (caller handles) | Error returned to caller |
| Asynchronous | 2 retries (configurable) | DLQ or OnFailure destination |
| Event source (SQS) | Visibility timeout retry | DLQ on source queue |
| Event source (Kinesis/DynamoDB) | Retry until expiry | Bisect on error, max retry attempts |

## Anti-hallucination rules

1. Always cite specific CloudWatch log excerpts, metrics, or API responses as evidence.
2. Memory controls CPU proportionally. Never claim CPU is independent of memory setting.
3. VPC Lambda needs NAT for internet. Never claim VPC Lambda has internet access by default.
4. Reserved concurrency is a CAP, not a guarantee. It limits max, doesn't pre-warm.
5. /tmp is ephemeral across cold starts. Never claim /tmp persists reliably.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 35 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Invocation Errors | A1-A4 | Timeout, OOM, unhandled exception, runtime crash |
| B — Cold Starts | B1-B3 | Init duration, VPC cold start, provisioned concurrency |
| C — Concurrency | C1-C3 | Throttling, reserved concurrency, account limits |
| D — Networking | D1-D4 | VPC no internet, VPC ENI limits, security group, DNS |
| E — Permissions | E1-E4 | Execution role, resource policy, KMS, cross-account |
| F — Deployment | F1-F3 | Package size, layer issues, container image |
| G — Event Sources | G1-G4 | SQS, Kinesis/DynamoDB Streams, API Gateway, S3 |
| H — Performance | H1-H3 | Memory/CPU tuning, I/O bottleneck, SDK initialization |
| I — Observability | I1-I2 | Missing logs, X-Ray tracing |
| Z — Catch-All | Z1 | General troubleshooting |
