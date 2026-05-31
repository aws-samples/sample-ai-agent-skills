---
name: edge-functions-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot CloudFront Functions
  and Lambda@Edge problems by analyzing function creation, deployment,
  replication, viewer/origin request/response errors, timeouts, memory
  limits, testing, logging, cache behavior association, and following
  structured runbooks.
  Activate when: CloudFront Function creation failures, publish errors,
  Lambda@Edge deployment issues, replication delays, viewer/origin
  event errors, timeout or memory issues, testing failures, logging
  problems, cache behavior association errors, or the user says
  something is wrong with edge functions.
compatibility: >
  Requires AWS CLI or SDK access with cloudfront, lambda, iam,
  cloudwatch, logs, and cloudtrail permissions.
---

# Edge Functions Diagnostics (CloudFront Functions & Lambda@Edge)

## When to use

Any CloudFront Functions or Lambda@Edge investigation — function creation, publishing, deployment, replication, viewer/origin request/response errors, timeouts, memory limits, testing, logging, or cache behavior association.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudfront list-functions --query 'FunctionList.Items[*].{Name:Name,Status:Status,Stage:FunctionConfig.Runtime}'
aws cloudfront list-distributions --query 'DistributionList.Items[*].{Id:Id,Status:Status,DomainName:DomainName}'
aws lambda list-functions --query 'Functions[?starts_with(FunctionName, `us-east-1.`)]'
```

### Step 2 — Domain deep dive

```
aws cloudfront describe-function --name <function-name> --stage LIVE
aws cloudfront get-distribution-config --id <distribution-id>
aws lambda get-function --function-name <function-name> --qualifier <version>
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id>,Name=FunctionName,Value=<func-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws logs filter-log-events --log-group-name /aws/cloudfront/function/<function-name> --start-time <epoch-ms> --filter-pattern "ERROR"
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudfront.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any edge function issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `cloudfront list-functions` | List CloudFront Functions |
| `cloudfront describe-function` | Get function details and metadata |
| `cloudfront test-function` | Test function with a test event |
| `cloudfront get-distribution-config` | Get distribution and cache behavior config |
| `lambda get-function` | Get Lambda@Edge function details |
| `cloudwatch get-metric-statistics` | Check function execution metrics |
| `logs filter-log-events` | Search function execution logs |

## Gotchas: Edge Functions

- CloudFront Functions run in the viewer-facing edge (viewer-request, viewer-response only). Lambda@Edge runs in regional edge caches (all four event types).
- CloudFront Functions have a 1ms execution time limit and 2MB max package size. Lambda@Edge has 5s (viewer) / 30s (origin) timeout and 1MB/50MB package limits.
- CloudFront Functions use JavaScript (ECMAScript 5.1 compatible runtime). Lambda@Edge supports Node.js and Python.
- Lambda@Edge functions MUST be deployed in us-east-1. They are replicated to edge locations automatically.
- Lambda@Edge functions cannot use environment variables, layers, VPC, DLQ, or provisioned concurrency.
- CloudFront Functions cannot make network calls. Lambda@Edge can make network calls in origin events.
- Function associations are per cache behavior, not per distribution. Different paths can use different functions.

## Anti-hallucination rules

1. Always cite specific function names, distribution IDs, or CloudWatch metrics as evidence.
2. CloudFront Functions and Lambda@Edge have different capabilities and limits. Never conflate them.
3. Lambda@Edge MUST be in us-east-1. Never suggest deploying in other regions.
4. CloudFront Functions have a 1ms limit. Never suggest complex processing in CloudFront Functions.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — CloudFront Functions | A1-A2 | Function creation, publish errors |
| B — Lambda@Edge Deployment | B1-B2 | Deployment, replication |
| C — Viewer Events | C1-C2 | Viewer request errors, viewer response errors |
| D — Origin Events | D1-D2 | Origin request errors, origin response errors |
| E — Resource Limits | E1-E2 | Function timeout, memory limits |
| F — Development | F1-F2 | Testing failures, logging issues |
| G — Association | G1 | Cache behavior association |
| Z — Catch-All | Z1 | General troubleshooting |
