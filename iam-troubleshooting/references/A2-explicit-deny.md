---
title: "A2 — Explicit Deny Statements"
description: "Find explicit deny statements across all policy types that block access"
status: active
severity: HIGH
triggers:
  - "explicit deny"
  - "deny overriding allow"
  - "policy deny"
  - "cannot override deny"
owner: devops-agent
objective: "Locate the explicit deny statement blocking access across identity, resource, SCP, boundary, and session policies"
context: "An explicit Deny in ANY policy always wins, regardless of any Allow statements elsewhere. Deny statements can exist in identity policies, resource policies, SCPs, permission boundaries, session policies, and VPC endpoint policies."
---

## Phase 1 — Triage

MUST:
- Identify the denied action and resource from CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<action>`
- Check all identity policies for Deny statements:
  - `aws iam list-attached-user-policies --user-name <user>`
  - `aws iam list-user-policies --user-name <user>`
  - `aws iam get-policy-version --policy-arn <arn> --version-id <version>` (look for "Effect": "Deny")
- Check resource-based policy for Deny: varies by service (e.g., `aws s3api get-bucket-policy`, `aws sqs get-queue-attributes`)
- Simulate to confirm deny: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names <action> --resource-arns <resource>`

SHOULD:
- Check SCPs for Deny statements: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY` then `aws organizations describe-policy --policy-id <id>`
- Check permission boundary for missing Allow (acts as implicit deny): `aws iam get-user --user-name <user>` or `aws iam get-role --role-name <role>`
- Check VPC endpoint policy if request is from VPC: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>`

MAY:
- Search all policies in the account for Deny statements: `aws iam get-account-authorization-details --filter User,Role,Group,LocalManagedPolicy,AWSManagedPolicy`
- Use Access Analyzer to find policy issues: `aws accessanalyzer list-findings --analyzer-arn <arn>`

## Phase 2 — Remediate

MUST:
- Remove or modify the explicit Deny statement if it is overly broad
- If the Deny is intentional (security guardrail), adjust the request to avoid triggering it
- Verify Deny conditions — many Deny statements use conditions (e.g., `StringNotEquals`, `IpAddress`) that may be too broad

SHOULD:
- Check Deny statement conditions carefully — the deny may only apply under specific conditions
- Use `aws iam simulate-principal-policy` with `--context-entries` to test condition evaluation
- Document the Deny statement and its purpose before modifying

MAY:
- Refine Deny statements with more specific conditions instead of removing them
- Add NotAction or NotResource to narrow the Deny scope

## Common Issues

- symptoms: "IAM policy allows the action but it's still denied"
  diagnosis: "An SCP, resource policy, or VPC endpoint policy has an explicit Deny."
  resolution: "Check all policy layers. Use CloudTrail errorCode and errorMessage for clues."

- symptoms: "Deny statement uses StringNotEquals condition and blocks unexpected principals"
  diagnosis: "Condition logic is inverted — StringNotEquals denies everyone NOT in the list."
  resolution: "Verify the condition values include all intended principals. Add missing values."

- symptoms: "Deny with aws:PrincipalOrgID blocks a service-linked role"
  diagnosis: "Service-linked roles may not have the expected org ID in their session context."
  resolution: "Add an exception for service-linked roles using aws:PrincipalIsAWSService condition."

## Output Format

```yaml
root_cause: "explicit_deny — <policy_type>: <deny_statement_summary>"
evidence:
  - type: deny_statement
    policy_source: "<policy_arn_or_type>"
    content: "<full Deny statement JSON>"
  - type: cloudtrail_event
    content: "<denied event with errorCode>"
severity: HIGH
mitigation:
  immediate: "Modify or remove the explicit Deny, or adjust the request"
  long_term: "Audit all Deny statements for overly broad scope"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws cloudtrail lookup-events` | GREEN | Read-only diagnostic |
| `aws iam list-attached-user-policies` / `list-user-policies` | GREEN | Read-only diagnostic |
| `aws iam get-policy-version` | GREEN | Read-only diagnostic |
| `aws iam simulate-principal-policy` | GREEN | Read-only diagnostic |
| `aws iam get-account-authorization-details` | GREEN | Read-only diagnostic |
| `aws ec2 describe-vpc-endpoints` | GREEN | Read-only diagnostic |
| `aws accessanalyzer list-findings` | GREEN | Read-only diagnostic |
| Remove or modify explicit Deny statement | RED | Removing a Deny can grant previously blocked access across multiple principals |
| Refine Deny with NotAction/NotResource | YELLOW | Narrowing Deny scope changes effective permissions |

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

- No temporary resources are created during explicit deny analysis

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
