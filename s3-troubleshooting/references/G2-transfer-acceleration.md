---
title: "G2 — Transfer Acceleration Issues"
description: "Diagnose S3 Transfer Acceleration configuration and performance issues"
status: active
severity: MEDIUM
triggers:
  - "Transfer Acceleration"
  - "Accelerate endpoint"
  - "Slow uploads"
  - "s3-accelerate"
owner: devops-agent
objective: "Identify and fix Transfer Acceleration configuration or performance issues"
context: "S3 Transfer Acceleration uses CloudFront edge locations to speed up long-distance transfers. It requires a DNS-compatible bucket name (no dots). The accelerate endpoint is <bucket>.s3-accelerate.amazonaws.com. It adds cost per GB transferred. It helps most for long-distance, large file transfers."
---

## Phase 1 — Triage

MUST:
- Check if Transfer Acceleration is enabled: `aws s3api get-bucket-accelerate-configuration --bucket <bucket>`
- Verify bucket name is DNS-compatible (no dots): bucket names with dots cannot use acceleration
- Verify the application uses the accelerate endpoint: `<bucket>.s3-accelerate.amazonaws.com`
- Test acceleration benefit: `aws s3api put-bucket-accelerate-configuration --bucket <bucket> --accelerate-configuration Status=Enabled`

SHOULD:
- Compare transfer speeds with and without acceleration
- Check if the transfer distance justifies acceleration (same-region transfers may not benefit)
- Verify the application/SDK is configured to use the accelerate endpoint

MAY:
- Use the S3 Transfer Acceleration speed comparison tool
- Check CloudWatch metrics for transfer performance

## Phase 2 — Remediate

MUST:
- Enable Transfer Acceleration on the bucket
- Configure the application to use the accelerate endpoint
- Ensure bucket name has no dots (periods)

SHOULD:
- Test with the speed comparison tool before committing to acceleration
- Use multipart upload with acceleration for large files
- Monitor costs — acceleration adds per-GB charges

MAY:
- Consider AWS Global Accelerator as an alternative for non-S3 workloads
- Use CloudFront for read acceleration instead of Transfer Acceleration

## Common Issues

- symptoms: "Transfer Acceleration not improving speed"
  diagnosis: "Client is close to the S3 region — acceleration adds overhead for short distances."
  resolution: "Acceleration helps for long-distance transfers. For same-region, use direct endpoint."

- symptoms: "Cannot enable Transfer Acceleration"
  diagnosis: "Bucket name contains dots (e.g., my.bucket.name)."
  resolution: "Create a new bucket without dots in the name."

- symptoms: "Application not using accelerate endpoint"
  diagnosis: "SDK or application is using the standard S3 endpoint."
  resolution: "Configure SDK with useAccelerateEndpoint=true or use the .s3-accelerate.amazonaws.com endpoint."

## Output Format

```yaml
root_cause: "transfer_acceleration — <specific_cause>"
evidence:
  - type: accelerate_config
    content: "<acceleration status>"
  - type: endpoint_used
    content: "<endpoint in application>"
severity: MEDIUM
mitigation:
  immediate: "Enable acceleration and configure the correct endpoint"
  long_term: "Evaluate cost-benefit and use multipart upload with acceleration"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses get-bucket-accelerate-configuration and transfer speed testing. Remediation enables Transfer Acceleration via put-bucket-accelerate-configuration, which is a performance optimization with no security impact. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Transfer Acceleration cost changes affect budget

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
