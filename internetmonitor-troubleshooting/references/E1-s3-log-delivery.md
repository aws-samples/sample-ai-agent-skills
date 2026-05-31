---
title: "E1 — S3 Log Delivery"
description: "Diagnose S3 log delivery failures for Internet Monitor"
status: active
severity: MEDIUM
triggers: ["S3 logs missing", "log delivery failed", "no logs in S3", "S3 delivery error"]
owner: devops-agent
objective: "Resolve S3 log delivery issues for Internet Monitor data export"
context: "Internet Monitor can deliver logs to S3 for long-term storage and analysis. Delivery must be explicitly configured and requires correct S3 bucket permissions."
---

## Phase 1 — Triage

MUST:
- Check monitor log delivery configuration: `aws internetmonitor get-monitor --monitor-name <name> --query 'InternetMeasurementsLogDelivery'`
- Verify S3 bucket exists: `aws s3api head-bucket --bucket <bucket-name>`
- Check S3 bucket policy allows Internet Monitor to write: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Verify the S3 bucket is in the same region as the monitor
- Check for S3 bucket encryption settings that may block writes
- Review CloudTrail for PutObject failures from the Internet Monitor service

MAY:
- Check S3 bucket access logging for delivery attempts
- Verify no S3 lifecycle rules are deleting logs prematurely

## Phase 2 — Remediate

MUST:
- Configure log delivery if not set: `aws internetmonitor update-monitor --monitor-name <name> --internet-measurements-log-delivery '{"S3Config":{"BucketName":"<bucket>","BucketPrefix":"<prefix>","LogDeliveryStatus":"ENABLED"}}'`
- Fix S3 bucket policy to allow Internet Monitor service principal to write objects
- Ensure bucket exists and is accessible

SHOULD:
- Set appropriate bucket prefix for log organization
- Configure S3 lifecycle rules for log retention management
- Enable S3 bucket versioning for log integrity

MAY:
- Set up S3 event notifications for log processing pipelines
- Configure cross-region replication for log durability

## Common Issues

- symptoms: "Log delivery configured but no objects appearing in S3 bucket"
  diagnosis: "S3 bucket policy may not grant write access to the Internet Monitor service."
  resolution: "Update bucket policy to allow s3:PutObject from the internetmonitor.amazonaws.com service principal. Verify the bucket name and prefix are correct."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "s3api head-bucket: GREEN — read-only bucket existence check"
  - "s3api get-bucket-policy: GREEN — read-only policy retrieval"
  - "update-monitor log delivery: YELLOW — enables data export to S3"
```

## Escalation Conditions

- S3 bucket policy is correct but logs still not delivered after 1 hour
- Log delivery causes S3 bucket to exceed storage budget
- Cross-account S3 delivery fails despite correct bucket policy
- Log format or content does not match expected schema

## Data Sensitivity

- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access permissions
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- Internet Monitor logs in S3: MEDIUM — contain traffic patterns and city-network data

## Prohibited Actions

- NEVER modify S3 bucket policies on production buckets without reviewing existing permissions
- NEVER delete existing S3 objects to troubleshoot delivery issues
- NEVER disable S3 bucket encryption to work around delivery failures
- NEVER grant overly broad S3 permissions (s3:*) to fix delivery issues

## Phase 3 — Rollback

1. If log delivery was enabled in error, disable it: `aws internetmonitor update-monitor --monitor-name <name> --internet-measurements-log-delivery '{"S3Config":{"LogDeliveryStatus":"DISABLED"}}'`
2. If S3 bucket policy was modified, revert to previous policy
3. If S3 lifecycle rules were changed, restore original rules
4. Verify no sensitive data was exposed through bucket policy changes
5. Document all S3 and monitor configuration changes

## Output Format

```yaml
root_cause: "s3_log_delivery — <specific_cause>"
evidence:
  - type: delivery_error
    content: "<S3 and delivery configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 delivery configuration"
  long_term: "Automate S3 bucket and delivery setup via IaC"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
