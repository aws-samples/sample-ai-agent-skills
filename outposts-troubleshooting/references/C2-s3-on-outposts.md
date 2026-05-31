---
title: "C2 — S3 on Outposts Issues"
description: "Diagnose S3 on Outposts bucket, access point, and data access problems"
status: active
severity: HIGH
triggers:
  - "S3 on Outposts"
  - "s3-outposts"
  - "outpost bucket"
  - "access point outpost"
owner: devops-agent
objective: "Resolve S3 on Outposts bucket creation, access point configuration, and data access issues"
context: "S3 on Outposts stores data locally on the Outpost. Access is exclusively through S3 access points — standard S3 URLs do not work. Bucket capacity is limited by local storage. Data does not replicate to the Region automatically."
---

## Phase 1 — Triage

MUST:
- List S3 on Outposts buckets: `aws s3control list-regional-buckets --account-id <account-id> --outpost-id <outpost-id>`
- Get bucket details: `aws s3control get-bucket --account-id <account-id> --bucket <bucket-arn>`
- List access points: `aws s3control list-access-points --account-id <account-id> --bucket <bucket-arn>`
- Get access point details: `aws s3control get-access-point --account-id <account-id> --name <access-point-arn>`

SHOULD:
- Check access point policy: `aws s3control get-access-point-policy --account-id <account-id> --name <access-point-arn>`
- Verify VPC endpoint configuration for S3 on Outposts
- Check IAM permissions for s3-outposts actions

MAY:
- Review bucket lifecycle configuration
- Check bucket tagging and metadata

## Phase 2 — Remediate

MUST:
- If access denied: verify access point policy and IAM permissions include s3-outposts actions
- If bucket creation fails: check local storage capacity and service link connectivity
- Ensure access is through access points, not standard S3 URLs

SHOULD:
- Configure access point policies with least-privilege access
- Set up VPC endpoints for S3 on Outposts access from the VPC
- Monitor local storage utilization

MAY:
- Implement lifecycle policies for data management
- Set up cross-Region data replication using application-level copy

## Common Issues

- symptoms: "Access denied when accessing S3 on Outposts bucket"
  diagnosis: "Using standard S3 URL instead of access point, or missing s3-outposts IAM permissions."
  resolution: "Use S3 access point ARN for access. Add s3-outposts:* permissions to IAM policy."

- symptoms: "Cannot create bucket on Outpost"
  diagnosis: "Local storage capacity exhausted or service link down."
  resolution: "Check storage capacity. Verify service link connectivity for bucket creation."

## Output Format

```yaml
root_cause: "s3_outposts — <specific_cause>"
evidence:
  - type: bucket_config
    content: "<bucket and access point configuration>"
  - type: access_policy
    content: "<access point policy>"
severity: HIGH
mitigation:
  immediate: "Fix access point configuration or permissions"
  long_term: "Implement proper access point policies and storage monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — S3 on Outposts stores application data locally
- Service Disruption Risk: MEDIUM — S3 issues affect data access but not compute
- Compliance Impact: HIGH — local S3 storage must meet data residency requirements

## Escalation Conditions
- S3 on Outposts becomes inaccessible
- Storage capacity approaching limits
- Access point configuration issues
- Bucket policy changes block legitimate access
- Data replication to Region fails

## Data Sensitivity
- Bucket contents — CONFIDENTIAL
- Bucket policies — INTERNAL
- Access point configurations — INTERNAL
- Endpoint configurations — INTERNAL
- Never expose bucket contents or access credentials

## Prohibited Actions
- Do NOT make S3 on Outposts buckets publicly accessible
- Do NOT disable server-side encryption
- Do NOT ignore storage capacity warnings
- Do NOT grant overly broad bucket policies
- Do NOT bypass access point restrictions
- Do NOT delete buckets without verifying data backup

## Phase 3 — Rollback
- Restore bucket policies to previous configuration
- Re-enable encryption if disabled
- Restore access point configurations
- Verify data accessibility after changes
- Restore deleted objects from backups if available
- Document S3 configuration changes and their impact

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
