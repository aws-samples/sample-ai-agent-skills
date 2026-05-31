---
title: "A1 — S3 AccessDenied Errors"
description: "Diagnose AccessDenied errors on S3 operations by evaluating the full policy chain"
status: active
severity: HIGH
triggers:
  - "AccessDenied"
  - "Access Denied"
  - "403 Forbidden"
  - "s3 permission denied"
owner: devops-agent
objective: "Identify which policy layer is denying access and fix the permission gap"
context: "S3 access evaluation checks IAM policy, bucket policy, ACLs, S3 Block Public Access, VPC endpoint policy, and SCPs. An explicit deny in ANY layer blocks access. The most common causes are missing IAM permissions, restrictive bucket policies, and Block Public Access overrides."
---

## Phase 1 — Triage

MUST:
- Identify the exact API call, bucket, key, and principal from the error or CloudTrail
- Check the bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check Block Public Access: `aws s3api get-public-access-block --bucket <bucket>`
- Check account-level Block Public Access: `aws s3control get-public-access-block --account-id <account-id>`
- Simulate the IAM policy: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/<key>`

SHOULD:
- Check object ACL if ACLs are enabled: `aws s3api get-object-acl --bucket <bucket> --key <key>`
- Check object ownership setting: `aws s3api get-bucket-ownership-controls --bucket <bucket>`
- Check VPC endpoint policy if the request originates from a VPC: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- Review CloudTrail for the denied event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GetObject`

MAY:
- Check SCPs on the account if in an AWS Organization
- Check S3 Access Analyzer for policy findings: `aws accessanalyzer list-findings --analyzer-arn <arn>`

## Phase 2 — Remediate

MUST:
- Fix the specific policy layer that is denying access
- For IAM: add the required s3 action to the principal's policy
- For bucket policy: add an Allow statement or remove the explicit Deny
- For Block Public Access: adjust settings if public access is intended

SHOULD:
- Use least-privilege — grant only the specific actions and resources needed
- Test with simulate-principal-policy after the fix
- Verify the fix does not open unintended access

MAY:
- Enable S3 server access logging for ongoing audit: `aws s3api put-bucket-logging --bucket <bucket> --bucket-logging-status '{"LoggingEnabled":{"TargetBucket":"<log-bucket>","TargetPrefix":"s3-logs/"}}'`

## Common Issues

- symptoms: "AccessDenied on GetObject but IAM policy allows s3:*"
  diagnosis: "Bucket policy has an explicit Deny that overrides the IAM Allow."
  resolution: "Check bucket policy for Deny statements. Explicit deny always wins."

- symptoms: "AccessDenied after enabling Block Public Access"
  diagnosis: "Block Public Access blocks bucket policies that grant public access."
  resolution: "If public access is not needed, fix the application to use authenticated requests. If needed, adjust Block Public Access settings."

- symptoms: "AccessDenied on PutObject with SSE-KMS"
  diagnosis: "Missing kms:GenerateDataKey permission on the KMS key."
  resolution: "Add kms:GenerateDataKey to the IAM policy and ensure the KMS key policy allows the principal."

## Output Format

```yaml
root_cause: "access_denied — <policy_layer>: <specific_cause>"
evidence:
  - type: bucket_policy
    content: "<relevant policy statement>"
  - type: iam_simulation
    content: "<simulate-principal-policy result>"
severity: HIGH
mitigation:
  immediate: "Fix the denying policy layer"
  long_term: "Implement S3 Access Analyzer and server access logging"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟢 GREEN | Primarily diagnostic — uses get-bucket-policy, get-public-access-block, simulate-principal-policy, head-object. Remediation targets the specific denying policy layer with least-privilege fixes. |

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
