---
title: "E2 — Redshift Destination Errors"
description: "Diagnose Amazon AppFlow Redshift destination write failures"
status: active
severity: HIGH
triggers: ["Redshift destination error", "Redshift COPY failed", "Redshift load error", "staging bucket error"]
owner: devops-agent
objective: "Resolve Redshift destination issues including staging bucket, COPY command, IAM role, and schema errors"
context: "AppFlow uses intermediate S3 staging for Redshift. Data is written to S3 first, then loaded via COPY. Requires both S3 and Redshift permissions."
---

## Phase 1 — Triage

MUST:
- Check flow destination config: `aws appflow describe-flow --flow-name <flow-name>`
- Verify Redshift cluster is available: `aws redshift describe-clusters --cluster-identifier <cluster-id> --query 'Clusters[0].{Status:ClusterStatus,Endpoint:Endpoint}'`
- Check execution records for Redshift errors: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 5`

SHOULD:
- Verify intermediate S3 staging bucket exists and has correct permissions
- Check Redshift IAM role has COPY permissions on the staging bucket

MAY:
- Review Redshift system tables for COPY errors: `SELECT * FROM stl_load_errors ORDER BY starttime DESC LIMIT 10;`
- Check Redshift cluster security group allows inbound connections

## Phase 2 — Remediate

MUST:
- Fix intermediate S3 bucket permissions for AppFlow and Redshift
- Ensure Redshift IAM role has s3:GetObject, s3:ListBucket on staging bucket
- Verify Redshift table schema matches AppFlow field mappings

SHOULD:
- Test with a small dataset to verify end-to-end connectivity
- Verify Redshift cluster has sufficient storage

MAY:
- Optimize Redshift table sort keys and distribution for AppFlow loads

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "redshift describe-clusters: GREEN — read-only cluster info"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "Redshift COPY role modification: RED — affects data loading permissions"
```

## Escalation Conditions

- Redshift cluster is in a different VPC and requires networking changes
- Staging bucket permissions affect other data pipelines
- Redshift table schema changes require DBA approval
- COPY failures cause data inconsistency in Redshift tables

## Data Sensitivity

- `aws redshift describe-clusters`: MEDIUM — reveals cluster configuration and endpoint
- `aws appflow describe-flow`: MEDIUM — reveals destination configuration
- Redshift stl_load_errors: HIGH — may contain data samples from failed loads

## Prohibited Actions

- NEVER grant Redshift IAM role s3:* permissions as a workaround
- NEVER modify Redshift table schema without coordinating with data consumers
- NEVER delete staging bucket data before confirming successful COPY completion
- NEVER change Redshift cluster security groups without network team approval

## Phase 3 — Rollback

1. If Redshift table data was partially loaded, delete incomplete records using execution timestamp
2. If staging bucket policy was changed, revert to previous policy
3. If Redshift IAM role was modified, restore original role permissions
4. Clean up staging bucket files from failed executions
5. Verify Redshift table data integrity after rollback

## Common Issues

- symptoms: "Flow execution fails with 'S3ServiceException: Access Denied' during Redshift load"
  diagnosis: "Redshift IAM role does not have s3:GetObject on the intermediate staging bucket."
  resolution: "Update Redshift IAM role to include s3:GetObject and s3:ListBucket on the staging bucket ARN."

## Output Format

```yaml
root_cause: "redshift_destination — <specific_cause>"
evidence:
  - type: redshift_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Redshift/S3 permissions"
  long_term: "Automate permission validation"
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
