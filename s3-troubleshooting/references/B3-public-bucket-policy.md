---
title: "B3 — Public Bucket Policy Issues"
description: "Diagnose unintended public access through bucket policies"
status: active
severity: MEDIUM
triggers:
  - "Public bucket"
  - "Unintended public access"
  - "Bucket is public"
  - "Public policy"
owner: devops-agent
objective: "Identify and remediate unintended public access granted through bucket policies"
context: "A bucket policy with Principal '*' and no restrictive conditions grants public access. S3 Access Analyzer and Block Public Access help detect and prevent this. Unintended public access is a common security finding."
---

## Phase 1 — Triage

MUST:
- Check the bucket policy for public statements: `aws s3api get-bucket-policy --bucket <bucket>`
- Look for Principal "*" or Principal {"AWS": "*"} without conditions
- Check Block Public Access settings: `aws s3api get-public-access-block --bucket <bucket>`
- Check S3 Access Analyzer: `aws accessanalyzer list-findings --analyzer-arn <arn>`

SHOULD:
- Verify if the public access is intentional (static website, public datasets)
- Check for overly broad conditions (e.g., aws:SourceIp with wide CIDR)
- Review CloudTrail for who added the public policy statement

MAY:
- Use IAM Access Analyzer policy validation to check for public access
- Check AWS Config rules for s3-bucket-public-read-prohibited and s3-bucket-public-write-prohibited

## Phase 2 — Remediate

MUST:
- If unintended: remove the public statement from the bucket policy
- Enable Block Public Access to prevent future public policies
- If intended: document the justification and add restrictive conditions

SHOULD:
- Replace public access with CloudFront OAC for web content
- Use presigned URLs for temporary access instead of public policies
- Enable S3 Access Analyzer for ongoing monitoring

MAY:
- Implement SCPs to prevent public bucket policies in the organization
- Set up AWS Config auto-remediation for public bucket findings

## Common Issues

- symptoms: "S3 Access Analyzer reports bucket as public"
  diagnosis: "Bucket policy has Principal * without restrictive conditions."
  resolution: "Add conditions (aws:SourceIp, aws:SourceVpce, aws:PrincipalOrgID) or remove the public statement."

- symptoms: "Cannot remove public access — application depends on it"
  diagnosis: "Application serves content directly from S3 public URL."
  resolution: "Migrate to CloudFront with OAC. Update application URLs to use CloudFront domain."

- symptoms: "Bucket policy looks private but Access Analyzer says public"
  diagnosis: "A condition in the policy is too broad (e.g., wide IP range, wildcard in StringLike)."
  resolution: "Tighten conditions. Access Analyzer evaluates policy logic, not just Principal."

## Output Format

```yaml
root_cause: "public_bucket_policy — <specific_cause>"
evidence:
  - type: bucket_policy
    content: "<public statement>"
  - type: access_analyzer
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Remove or restrict the public policy statement"
  long_term: "Enable Block Public Access and S3 Access Analyzer"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🔴 RED | Directly deals with public bucket policies and may involve modifying Block Public Access settings. Remediation of unintended public access is critical; incorrect changes can expose or break access. |

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
