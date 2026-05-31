---
title: "A2 — Bucket Policy Conflicts"
description: "Diagnose conflicts and unexpected denials caused by bucket policy evaluation"
status: active
severity: HIGH
triggers:
  - "Bucket policy conflict"
  - "Policy evaluation unexpected"
  - "Explicit deny in bucket policy"
  - "Bucket policy size limit"
owner: devops-agent
objective: "Identify conflicting statements in the bucket policy and resolve the evaluation issue"
context: "Bucket policies follow IAM policy evaluation logic: explicit deny overrides any allow. Policies have a 20 KB size limit. Common issues include conflicting Deny/Allow statements, incorrect principal formats, and condition key mismatches."
---

## Phase 1 — Triage

MUST:
- Get the bucket policy: `aws s3api get-bucket-policy --bucket <bucket> --output text | python3 -m json.tool`
- Check policy size (20 KB limit): `aws s3api get-bucket-policy --bucket <bucket> --output text | wc -c`
- Identify all Deny statements and their conditions
- Simulate the request: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*`

SHOULD:
- Check for NotPrincipal usage (often causes unintended denials)
- Verify Resource ARN format (bucket ARN vs object ARN: `arn:aws:s3:::bucket` vs `arn:aws:s3:::bucket/*`)
- Check for condition key mismatches (aws:SourceIp, aws:SourceVpce, aws:PrincipalOrgID)

MAY:
- Use IAM Access Analyzer to validate the policy: `aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type RESOURCE_POLICY`
- Check CloudTrail for the specific denied request details

## Phase 2 — Remediate

MUST:
- Remove or fix conflicting Deny statements
- Ensure Resource ARN matches the intended scope (bucket-level vs object-level)
- Fix principal format (account ID vs ARN vs wildcard)

SHOULD:
- Consolidate overlapping statements to reduce policy size
- Use conditions instead of multiple statements where possible
- Test the updated policy with simulate-principal-policy

MAY:
- Split complex policies across IAM and bucket policy to stay under 20 KB
- Document the policy intent with Sid fields for each statement

## Common Issues

- symptoms: "All access denied despite Allow statement"
  diagnosis: "A Deny statement with broad scope overrides the Allow."
  resolution: "Check for Deny with Resource arn:aws:s3:::bucket/* or Principal *. Narrow the Deny scope."

- symptoms: "Policy update fails with MalformedPolicy"
  diagnosis: "Policy exceeds 20 KB or has syntax errors."
  resolution: "Reduce policy size by consolidating statements. Validate JSON syntax."

- symptoms: "Some principals denied but not others"
  diagnosis: "NotPrincipal in a Deny statement denies everyone except the listed principals."
  resolution: "Replace NotPrincipal with explicit Condition keys like aws:PrincipalArn."

## Output Format

```yaml
root_cause: "bucket_policy_conflict — <specific_conflict>"
evidence:
  - type: bucket_policy
    content: "<conflicting statements>"
  - type: policy_simulation
    content: "<simulation result>"
severity: HIGH
mitigation:
  immediate: "Fix the conflicting policy statements"
  long_term: "Use IAM Access Analyzer policy validation and document policy intent with Sid"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying bucket policies to resolve conflicts. Policy changes can inadvertently grant or deny access to unintended principals. Uses put-bucket-policy to fix conflicting statements. |

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
