---
title: "G1 — S3 503 SlowDown Throttling"
description: "Diagnose S3 request throttling (503 SlowDown) and per-prefix rate limits"
status: active
severity: HIGH
triggers:
  - "503 SlowDown"
  - "S3 throttling"
  - "Request rate too high"
  - "Reduce your request rate"
owner: devops-agent
objective: "Identify the cause of S3 throttling and implement request distribution strategies"
context: "S3 supports 5,500 GET/HEAD and 3,500 PUT/POST/DELETE requests per second per prefix. Exceeding these limits causes 503 SlowDown errors. S3 automatically partitions by prefix. Spreading requests across multiple prefixes increases aggregate throughput. Retry with exponential backoff is essential."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch 5xx metrics: `aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name 5xxErrors --dimensions Name=BucketName,Value=<bucket> Name=FilterId,Value=EntireBucket --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Identify the request pattern: which prefixes, what operations, what rate
- Check S3 request metrics (must be enabled): `aws s3api get-bucket-metrics-configuration --bucket <bucket> --id <metrics-id>`
- Verify the application implements retry with exponential backoff

SHOULD:
- Analyze the key naming pattern for hot prefixes
- Check if the workload is read-heavy or write-heavy
- Review application logs for 503 responses and retry behavior

MAY:
- Enable S3 request metrics per prefix to identify hot spots
- Check if S3 Transfer Acceleration or CloudFront can offload reads

## Phase 2 — Remediate

MUST:
- Implement exponential backoff with jitter in the application
- Distribute requests across multiple prefixes if hitting per-prefix limits
- Use the AWS SDK's built-in retry mechanism (it handles 503 automatically)

SHOULD:
- Use CloudFront for read-heavy workloads to cache and reduce S3 requests
- Spread keys across prefixes: `prefix-1/`, `prefix-2/`, etc.
- Enable S3 request metrics to monitor per-prefix request rates

MAY:
- Use S3 Intelligent-Tiering with frequent access tier for hot data
- Consider S3 Express One Zone for latency-sensitive, high-request-rate workloads
- Pre-warm prefixes by gradually increasing request rate

## Common Issues

- symptoms: "503 SlowDown during batch processing"
  diagnosis: "Batch job sends too many requests to a single prefix."
  resolution: "Distribute objects across prefixes and parallelize across them."

- symptoms: "Intermittent 503 errors during peak hours"
  diagnosis: "Request rate exceeds per-prefix limits during peaks."
  resolution: "Implement retry with backoff. Consider CloudFront for reads."

- symptoms: "503 errors after migrating from many small files to fewer large files"
  diagnosis: "All requests now hit the same prefix, concentrating load."
  resolution: "Redistribute objects across multiple prefixes."

## Output Format

```yaml
root_cause: "throttling — <specific_cause>"
evidence:
  - type: cloudwatch_5xx
    content: "<5xx error count and pattern>"
  - type: request_pattern
    content: "<prefix distribution and request rate>"
severity: HIGH
mitigation:
  immediate: "Implement retry with exponential backoff"
  long_term: "Distribute requests across prefixes and use CloudFront for reads"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses CloudWatch metrics, get-bucket-metrics-configuration, and application log analysis. Remediation focuses on application-side changes (retry logic, prefix distribution, CloudFront caching), not bucket security controls. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Throttling affects production workloads with SLA requirements

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Lifecycle rules reveal data retention strategy"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
