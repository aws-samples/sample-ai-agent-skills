---
title: "B1 — Block Public Access Settings"
description: "Diagnose issues caused by S3 Block Public Access at account and bucket level"
status: active
severity: HIGH
triggers:
  - "Block Public Access"
  - "Cannot make bucket public"
  - "Public access blocked"
  - "BlockPublicAccess"
owner: devops-agent
objective: "Identify which Block Public Access setting is blocking access and determine the correct configuration"
context: "S3 Block Public Access has four independent settings at both account and bucket level. Account-level settings OVERRIDE bucket-level settings. New accounts have all four settings enabled by default. The four settings are: BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets."
---

## Phase 1 — Triage

MUST:
- Check account-level settings: `aws s3control get-public-access-block --account-id <account-id>`
- Check bucket-level settings: `aws s3api get-public-access-block --bucket <bucket>`
- Identify which of the four settings is blocking the intended access
- Understand the four settings:
  - BlockPublicAcls: rejects PUT requests that include public ACLs
  - IgnorePublicAcls: ignores all public ACLs on the bucket and objects
  - BlockPublicPolicy: rejects bucket policies that grant public access
  - RestrictPublicBuckets: restricts public bucket access to AWS services and authorized users only

SHOULD:
- Check if the bucket actually needs public access (most don't)
- Verify if CloudFront OAC/OAI can replace public bucket access
- Check the bucket policy for public statements: Principal "*" without conditions

MAY:
- Review S3 Access Analyzer findings for public access: `aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"resourceType":{"eq":["AWS::S3::Bucket"]}}'`

## Phase 2 — Remediate

MUST:
- If public access is NOT needed: keep Block Public Access enabled and fix the application
- If public access IS needed: disable the specific setting at BOTH account and bucket level
- Document the business justification for any public access

SHOULD:
- Use CloudFront with OAC instead of making the bucket public
- Enable S3 Access Analyzer to monitor for unintended public access
- Use presigned URLs for temporary public access instead of bucket policies

MAY:
- Set up CloudWatch Events for Block Public Access changes
- Implement SCPs to prevent disabling Block Public Access in production accounts

## Common Issues

- symptoms: "Bucket-level Block Public Access is disabled but bucket is still not public"
  diagnosis: "Account-level Block Public Access is enabled and overrides bucket settings."
  resolution: "Disable the specific account-level setting. Account settings always override."

- symptoms: "Cannot save a bucket policy that grants public access"
  diagnosis: "BlockPublicPolicy is enabled and rejects policies with public access."
  resolution: "Disable BlockPublicPolicy at both account and bucket level, or redesign without public access."

- symptoms: "Public ACLs are being ignored"
  diagnosis: "IgnorePublicAcls is enabled, which makes all public ACLs ineffective."
  resolution: "Disable IgnorePublicAcls if public ACL access is intended, or use bucket policy instead."

## Output Format

```yaml
root_cause: "block_public_access — <setting>: <account_or_bucket_level>"
evidence:
  - type: account_bpa
    content: "<account-level settings>"
  - type: bucket_bpa
    content: "<bucket-level settings>"
severity: HIGH
mitigation:
  immediate: "Adjust the specific Block Public Access setting"
  long_term: "Use CloudFront OAC instead of public buckets where possible"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🔴 RED | Directly involves modifying Block Public Access settings via put-public-access-block. Disabling these settings can expose buckets to public internet access. Highest-risk S3 security operation. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Bucket contains sensitive/regulated data (PII, PHI, financial)
- Cross-account access changes are needed
- Encryption configuration changes affect multiple consumers

## Rollback
1. Before any bucket policy change: Save current policy with `aws s3api get-bucket-policy`
2. Before ACL changes: Save current ACL with `aws s3api get-bucket-acl`
3. After change: Verify access works without granting excessive permissions
4. If change causes issues: Restore the saved policy/ACL immediately
5. Cleanup: Remove any temporary access grants

## Data Sensitivity
| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-bucket-policy` | HIGH | Contains access rules — redact principals |
| `get-bucket-acl` | MEDIUM | Shows grantees — summarize |
| `get-public-access-block` | MEDIUM | Security posture — safe to include |
| `list-objects` | LOW | Object keys only — safe to include |

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` in bucket policy without Condition keys
- NEVER suggest removing bucket encryption to fix access issues
- NEVER suggest making a bucket public to resolve CORS or access issues
- NEVER suggest `s3:*` in any IAM or bucket policy fix
- ALWAYS use least-privilege: grant only the specific S3 action needed
- ALWAYS check both account-level AND bucket-level Block Public Access

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
