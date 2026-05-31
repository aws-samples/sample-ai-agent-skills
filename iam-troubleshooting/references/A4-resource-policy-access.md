---
title: "A4 — Resource Policy Access"
description: "Diagnose access issues caused by resource-based policies granting or denying access independently"
status: active
severity: MEDIUM
triggers:
  - "resource policy"
  - "bucket policy"
  - "key policy"
  - "queue policy"
  - "resource-based policy"
owner: devops-agent
objective: "Determine if a resource-based policy is granting, denying, or failing to grant access and fix accordingly"
context: "Resource-based policies (S3 bucket policies, KMS key policies, SQS queue policies, etc.) are evaluated alongside identity policies. For same-account access, a resource policy Allow can grant access even without an identity policy Allow. For cross-account access, BOTH the resource policy and identity policy must allow."
---

## Phase 1 — Triage

MUST:
- Identify the resource and its policy:
  - S3: `aws s3api get-bucket-policy --bucket <bucket>`
  - KMS: `aws kms get-key-policy --key-id <key-id> --policy-name default`
  - SQS: `aws sqs get-queue-attributes --queue-url <url> --attribute-names Policy`
  - SNS: `aws sns get-topic-attributes --topic-arn <arn>`
  - Lambda: `aws lambda get-policy --function-name <function>`
- Check if the request is same-account or cross-account
- Verify the Principal in the resource policy matches the caller: `aws sts get-caller-identity`

SHOULD:
- Check for Deny statements in the resource policy that override Allow
- Verify condition keys in the resource policy (e.g., aws:SourceVpc, aws:SourceAccount)
- For KMS: check both key policy and IAM policy — KMS requires key policy to delegate to IAM

MAY:
- Use IAM Policy Simulator with resource policy context
- Check if the resource policy uses aws:PrincipalOrgID for organization-wide access

## Phase 2 — Remediate

MUST:
- For same-account: ensure either the resource policy OR identity policy allows the action
- For cross-account: ensure BOTH the resource policy AND identity policy allow the action
- For KMS: ensure the key policy has the `"kms:*"` delegation statement or explicitly allows the principal

SHOULD:
- Use specific principal ARNs instead of wildcards in resource policies
- Add conditions to restrict access (aws:SourceAccount, aws:PrincipalOrgID)
- Test access after changes

MAY:
- Use AWS RAM for cross-account resource sharing instead of resource policies
- Enable Access Analyzer to monitor resource policy changes

## Common Issues

- symptoms: "Cross-account access denied even though resource policy allows the account"
  diagnosis: "The calling account's identity policy does not have the corresponding Allow."
  resolution: "Add the permission to the caller's identity policy. Cross-account requires both sides."

- symptoms: "KMS access denied even though IAM policy allows kms:Decrypt"
  diagnosis: "KMS key policy does not delegate permissions to IAM. KMS requires explicit key policy."
  resolution: "Add the IAM delegation statement to the key policy or explicitly allow the principal in the key policy."

- symptoms: "Resource policy uses Principal: * but access is still denied"
  diagnosis: "An SCP, VPC endpoint policy, or Block Public Access setting is overriding the resource policy."
  resolution: "Check all policy layers. Principal: * is very broad and may be blocked by guardrails."

## Output Format

```yaml
root_cause: "resource_policy — <resource_type>: <specific_cause>"
evidence:
  - type: resource_policy
    content: "<relevant policy statement>"
  - type: caller_identity
    content: "<sts get-caller-identity output>"
severity: MEDIUM
mitigation:
  immediate: "Fix the resource policy or identity policy as needed"
  long_term: "Use Access Analyzer to monitor resource policy exposure"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws s3api get-bucket-policy` | GREEN | Read-only diagnostic |
| `aws kms get-key-policy` | GREEN | Read-only diagnostic |
| `aws sqs get-queue-attributes` | GREEN | Read-only diagnostic |
| `aws sns get-topic-attributes` | GREEN | Read-only diagnostic |
| `aws lambda get-policy` | GREEN | Read-only diagnostic |
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws s3api put-bucket-policy` | YELLOW | Modifies S3 bucket access — can grant or revoke cross-account access |
| `aws sqs set-queue-attributes` (Policy) | YELLOW | Modifies SQS queue access |
| `aws kms put-key-policy` | RED | KMS key policy changes can permanently lock out all access |
| Resource policy with `"Principal": "*"` | RED | Grants public access to the resource |

## Escalation Conditions

- Remediation requires modifying IAM policies in a production account
- Fix involves granting cross-account access
- Root cause involves the management account
- Multiple principals are affected
- The denied action involves security-sensitive services (KMS, STS, Organizations)

## Rollback

1. **Before any change**: Document current state with `aws iam get-policy-version` or `aws iam get-role`
2. **After change**: Verify the fix resolves the issue without granting excessive permissions
3. **If change causes issues**: Revert to the previous policy version or detach the modified policy
4. **Cleanup**: Remove any temporary policies or test principals created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-account-authorization-details` | HIGH | Contains all IAM entities — redact from stored output |
| `get-policy-version` | HIGH | Contains full policy document — summarize, don't reproduce |
| `simulate-principal-policy` | MEDIUM | Shows effective permissions — summarize results |
| `list-*` | LOW | Resource listings — safe to include |

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest disabling MFA requirements as a troubleshooting step
- ALWAYS identify the specific denied action from CloudTrail before suggesting a fix
- ALWAYS use least-privilege: grant only the specific action on the specific resource

## Cleanup

- No temporary resources are created during resource policy analysis

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
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
