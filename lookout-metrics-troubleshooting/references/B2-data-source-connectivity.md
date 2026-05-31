---
title: "B2 — Data Source Connectivity"
description: "Diagnose and resolve data source connectivity issues for Lookout for Metrics"
status: active
severity: HIGH
triggers: ["data source unreachable", "cannot connect to data source", "S3 access denied", "CloudWatch no data"]
owner: devops-agent
objective: "Establish reliable connectivity between the detector and its configured data source"
context: "Lookout for Metrics supports S3, CloudWatch, Redshift, and RDS data sources. Each has unique connectivity requirements including IAM permissions, network access, and data format expectations."
---

## Phase 1 — Triage

MUST:
- Describe metric set to identify data source type: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- For S3: verify bucket exists and IAM role has access: `aws s3 ls s3://<bucket>/<prefix>/`
- For CloudWatch: verify metrics exist: `aws cloudwatch list-metrics --namespace <namespace> --metric-name <name>`

SHOULD:
- Check the IAM execution role trust policy includes lookoutmetrics.amazonaws.com
- Verify the IAM role has the required data source permissions
- Check data quality metrics for connectivity-related failures: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`

MAY:
- Test cross-account access if data source is in a different account
- Review VPC configuration for Redshift/RDS data sources

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for the specific data source type
- For S3: ensure `s3:GetObject`, `s3:ListBucket` on the bucket and prefix
- For CloudWatch: ensure `cloudwatch:GetMetricData`, `cloudwatch:ListMetrics`

SHOULD:
- Update S3 bucket policy if cross-account access is needed
- Verify KMS key permissions if S3 objects are encrypted
- For Redshift/RDS: verify security group allows inbound from Lookout for Metrics

MAY:
- Set up VPC endpoints for private data source access
- Configure cross-account IAM roles for multi-account data sources

## Safety Ratings

```
safety_ratings:
  - "describe-metric-set: GREEN — read-only configuration check"
  - "s3 ls: GREEN — read-only bucket listing"
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "IAM role policy updates: YELLOW — modifies permissions"
```

## Escalation Conditions

- Data source is in a different account and cross-account role setup requires organization admin
- S3 bucket policy is managed by a different team and cannot be modified
- VPC configuration changes require network team approval
- KMS key policy changes require security team approval

## Data Sensitivity

- `aws lookoutmetrics describe-metric-set`: MEDIUM — contains data source connection details
- `aws s3 ls`: MEDIUM — reveals bucket contents and structure
- `aws cloudwatch list-metrics`: LOW — metric names and dimensions only
- IAM role policies: MEDIUM — reveals permission scope and trusted principals

## Prohibited Actions

- NEVER grant `s3:*` or `cloudwatch:*` permissions — use least-privilege
- NEVER modify S3 bucket policies without understanding the impact on other consumers
- NEVER expose database credentials in logs, outputs, or error messages
- NEVER open security groups to 0.0.0.0/0 for Redshift/RDS connectivity

## Phase 3 — Rollback

1. If IAM permissions were broadened for testing, revert to least-privilege: save original policy before changes
2. If S3 bucket policy was modified, revert to previous version
3. If security group rules were added, remove test rules after resolution
4. Document all connectivity changes and their justification
5. Verify other services using the same data source are not affected by changes

## Common Issues

- symptoms: "S3 data source returns AccessDenied"
  diagnosis: "IAM role lacks s3:GetObject or s3:ListBucket on the configured bucket/prefix."
  resolution: "Add S3 read permissions to the execution role. Check for bucket policy denials and KMS key access."

- symptoms: "CloudWatch data source returns no data points"
  diagnosis: "Metric namespace, name, or dimensions do not match exactly."
  resolution: "Verify with `aws cloudwatch list-metrics --namespace <ns>` and correct the metric set configuration."

## Output Format

```yaml
root_cause: "data_source_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details and data source configuration>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity issue"
  long_term: "Implement connectivity monitoring and alerting"
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
