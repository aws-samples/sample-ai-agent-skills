---
name: s3-objectlambda-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot S3 Object Lambda
  problems by analyzing access point creation, configuration, Lambda
  invocation errors, transformation failures, permission issues,
  performance problems, range/HEAD requests, CloudFront integration,
  and following structured runbooks.
  Activate when: Object Lambda access point creation failures, Lambda
  invocation errors, transformation failures, permission errors,
  timeout issues, range GET problems, HEAD object issues, CloudFront
  integration problems, or the user says something is wrong with
  S3 Object Lambda.
compatibility: >
  Requires AWS CLI or SDK access with s3, s3-object-lambda, lambda,
  iam, cloudfront, cloudwatch, and cloudtrail permissions.
---

# S3 Object Lambda Diagnostics

## When to use

Any S3 Object Lambda investigation — access point creation, configuration, Lambda invocation errors, transformation failures, permission issues, performance tuning, range/HEAD request handling, or CloudFront integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws s3control list-access-points-for-object-lambda --account-id <account-id>
aws s3control get-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name>
aws cloudwatch get-metric-statistics --namespace AWS/S3ObjectLambda --metric-name AllRequests --dimensions Name=AccessPointName,Value=<olap-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws lambda get-function --function-name <function-name>
aws s3control get-access-point-for-object-lambda --account-id <account-id> --name <olap-name>
aws lambda invoke --function-name <function-name> --payload '{"xAmzRequestId":"test","getObjectContext":{"inputS3Url":"https://s3.amazonaws.com/...","outputRoute":"...","outputToken":"..."}}' output.json
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=s3-object-lambda.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --start-time <epoch-ms> --filter-pattern "ERROR"
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function-name> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

Read `references/guardrails.md` before concluding on any S3 Object Lambda issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `s3control list-access-points-for-object-lambda` | List Object Lambda access points |
| `s3control get-access-point-for-object-lambda` | Get OLAP details |
| `s3control get-access-point-configuration-for-object-lambda` | Get OLAP configuration |
| `lambda get-function` | Get Lambda function details |
| `lambda invoke` | Test Lambda function directly |
| `cloudwatch get-metric-statistics` | Check OLAP and Lambda metrics |
| `logs filter-log-events` | Search Lambda execution logs |

## Gotchas: S3 Object Lambda

- Object Lambda Access Points (OLAPs) require a supporting S3 access point — you cannot attach them directly to a bucket.
- The Lambda function MUST call WriteGetObjectResponse to return transformed data. Failure to call this results in a 500 error to the caller.
- Object Lambda has a 60-second timeout for Lambda execution. This is separate from the Lambda function's own timeout setting.
- Range GET and part-number requests are forwarded to the Lambda function. The function must handle these explicitly or return the full object.
- HEAD object requests can be routed through Object Lambda. The function receives a HeadObject event type and must respond accordingly.
- OLAPs do NOT support anonymous access. All requests must be authenticated with SigV4.
- CloudFront can use OLAPs as origins, but requires OAC (Origin Access Control) — OAI is not supported.

## Anti-hallucination rules

1. Always cite specific OLAP ARNs, Lambda function names, or CloudWatch metrics as evidence.
2. Object Lambda Access Points require a supporting S3 access point. Never suggest attaching directly to a bucket.
3. The Lambda function MUST call WriteGetObjectResponse. Never omit this requirement.
4. Object Lambda has a 60-second execution limit. Never confuse this with the Lambda function's own timeout.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Access Points | A1-A2 | Creation, configuration |
| B — Lambda Execution | B1-B2 | Invocation errors, transformation failures |
| C — Permissions | C1-C2 | Permission errors, policy configuration |
| D — Performance | D1-D2 | Performance issues, timeout errors |
| E — Request Handling | E1-E2 | Range GET issues, HEAD object issues |
| F — Integration | F1 | CloudFront integration |
| Z — Catch-All | Z1 | General troubleshooting |
