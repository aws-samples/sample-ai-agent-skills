---
title: "Z1 — General S3 Troubleshooting (Catch-All)"
description: "Fallback SOP for S3 issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown S3 issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Identify the bucket: `aws s3api get-bucket-location --bucket <bucket>`
- Check bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check versioning: `aws s3api get-bucket-versioning --bucket <bucket>`
- Check encryption: `aws s3api get-bucket-encryption --bucket <bucket>`
- Check Block Public Access: `aws s3api get-public-access-block --bucket <bucket>`
- Check recent errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<bucket> --max-results 10`

SHOULD:
- Check replication: `aws s3api get-bucket-replication --bucket <bucket>`
- Check lifecycle: `aws s3api get-bucket-lifecycle-configuration --bucket <bucket>`
- Check notifications: `aws s3api get-bucket-notification-configuration --bucket <bucket>`
- Check CORS: `aws s3api get-bucket-cors --bucket <bucket>`
- Check CloudWatch S3 metrics for anomalies

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Access denied errors → Access & Permissions (A1-A4)
- Public access issues → Public Access (B1-B3)
- Encryption errors → Encryption (C1-C3)
- Replication failures → Replication (D1-D3)
- Lifecycle issues → Lifecycle (E1-E3)
- Versioning confusion → Versioning (F1-F3)
- Performance problems → Performance (G1-G3)
- Event/integration failures → Events & Integration (H1-H3)
- Website/CORS issues → Static Hosting & CORS (I1-I3)
- Compliance/lock issues → Object Lock (J1-J2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all bucket configurations systematically
- Review CloudTrail for recent configuration changes
- Check S3 server access logs if enabled
- Verify IAM permissions for the affected principal

SHOULD:
- Check AWS Health Dashboard for S3 service events
- Compare with a known-good bucket configuration
- Check S3 Storage Lens for anomalies

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<access|public_access|encryption|replication|lifecycle|versioning|performance|events|hosting|compliance|unknown>"
investigation_path: "bucket config → CloudTrail → metrics → <domain_classification>"
evidence:
  - type: bucket_config
    content: "<bucket configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses read-only commands (get-bucket-policy, get-bucket-versioning, get-bucket-encryption, get-public-access-block, get-bucket-location, CloudTrail lookup). No state-changing operations in the general triage phase. Remediations are deferred to specific runbooks. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Issue cannot be classified into a specific runbook category

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
