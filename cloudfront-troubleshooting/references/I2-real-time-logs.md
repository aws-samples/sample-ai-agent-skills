---
title: "I2 — Real-Time Log Issues"
description: "Diagnose CloudFront real-time log configuration and delivery issues"
status: active
severity: MEDIUM
triggers:
  - "real-time logs"
  - "real-time logging"
  - "Kinesis logs"
  - "log streaming"
owner: devops-agent
objective: "Resolve CloudFront real-time log delivery issues"
context: "CloudFront real-time logs deliver log records to Kinesis Data Streams within seconds. They support sampling (1-100% of requests) and field selection. Real-time logs require a Kinesis Data Stream in us-east-1. They are configured separately from standard access logs and can run simultaneously. Cost includes Kinesis Data Streams charges."
---

## Phase 1 — Triage

MUST:
- Check real-time log config: `aws cloudfront list-realtime-log-configs`
- Get specific config: `aws cloudfront get-realtime-log-config --name <config-name>`
- Check Kinesis stream status: `aws kinesis describe-stream --stream-name <stream-name> --region us-east-1`
- Verify the real-time log config is associated with a cache behavior

SHOULD:
- Check Kinesis stream shard count and throughput
- Verify IAM role allows CloudFront to write to Kinesis
- Check sampling rate configuration

MAY:
- Check Kinesis iterator age for delivery lag: `aws cloudwatch get-metric-statistics --namespace AWS/Kinesis --metric-name GetRecords.IteratorAgeMilliseconds --dimensions Name=StreamName,Value=<stream> --region us-east-1 --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check for Kinesis throttling

## Phase 2 — Remediate

MUST:
- Ensure Kinesis Data Stream is in us-east-1
- Configure IAM role with kinesis:PutRecord and kinesis:PutRecords permissions
- Associate real-time log config with the desired cache behavior(s)

SHOULD:
- Size Kinesis shards appropriately for traffic volume (1 MB/s or 1,000 records/s per shard)
- Use sampling for high-traffic distributions to control costs
- Select only needed fields to reduce data volume

MAY:
- Set up Kinesis Data Firehose for delivery to S3, Elasticsearch, or Redshift
- Implement real-time alerting based on log patterns

## Common Issues

- symptoms: "No records in Kinesis stream"
  diagnosis: "Real-time log config not associated with any cache behavior."
  resolution: "Associate the config with the desired behavior in distribution settings."

- symptoms: "Records delayed or missing"
  diagnosis: "Kinesis stream throttled due to insufficient shards."
  resolution: "Increase shard count to handle the request volume."

- symptoms: "Error creating real-time log config"
  diagnosis: "Kinesis stream not in us-east-1 or IAM role permissions insufficient."
  resolution: "Create stream in us-east-1 and verify IAM role permissions."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "real_time_logs — <specific_cause>"
evidence:
  - type: log_config
    content: "<real-time log configuration>"
  - type: kinesis_status
    content: "<Kinesis stream status and metrics>"
  - type: iam_role
    content: "<IAM role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix log configuration or Kinesis stream"
  long_term: "Implement proper stream sizing and monitoring"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
