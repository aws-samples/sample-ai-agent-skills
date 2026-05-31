---
title: "B4 — Chained Role Assumptions"
description: "Diagnose issues with role chaining, the 1-hour session limit, and credential forwarding"
status: active
severity: MEDIUM
triggers:
  - "chained role"
  - "role chaining"
  - "role assuming role"
  - "1 hour limit"
  - "credential forwarding"
owner: devops-agent
objective: "Identify role chaining issues and provide alternatives to work around the 1-hour session limit"
context: "Role chaining occurs when a role assumes another role. AWS limits chained role sessions to 1 hour regardless of MaxSessionDuration. The session also loses the original caller's identity — aws:SourceIdentity can preserve it. Chained roles cannot request longer sessions."
---

## Phase 1 — Triage

MUST:
- Identify the role chain: check CloudTrail for sequential AssumeRole events
- Verify the session is chained: look for `assumed-role` in the source ARN of the second AssumeRole call
- Check if the 1-hour limit is causing the issue: `aws sts get-caller-identity` (check session expiry)

SHOULD:
- Map the full chain: Role A → Role B → Role C (each hop adds 1-hour constraint)
- Check if aws:SourceIdentity is set: `aws sts get-caller-identity` (look for SourceIdentity)
- Verify each trust policy in the chain allows the previous role

MAY:
- Check if the chain can be simplified to a direct assumption
- Review if AWS SSO/Identity Center can replace the chain

## Phase 2 — Remediate

MUST:
- If the 1-hour limit is the problem, restructure to avoid chaining:
  - Have the original identity assume the final role directly
  - Use resource-based policies to grant access without role assumption
- Ensure each role in the chain has the correct trust policy for the previous role

SHOULD:
- Set aws:SourceIdentity to track the original caller through the chain: `aws sts assume-role --role-arn <arn> --role-session-name <name> --source-identity <original-user>`
- Use session tags to pass context through the chain
- Document the role chain architecture

MAY:
- Replace role chains with AWS SSO/Identity Center permission sets
- Use service-specific cross-account access (e.g., S3 bucket policy) instead of role chaining

## Common Issues

- symptoms: "Session expires after 1 hour even with MaxSessionDuration set to 12 hours"
  diagnosis: "This is a chained role session. The 1-hour limit is hard-coded for chained roles."
  resolution: "Assume the target role directly from the original identity, not through an intermediate role."

- symptoms: "Cannot determine who originally initiated the role chain"
  diagnosis: "Role chaining loses the original caller identity. Only the immediate previous role is visible."
  resolution: "Use aws:SourceIdentity to preserve the original caller through the chain."

- symptoms: "Third role in chain fails to assume"
  diagnosis: "Each role's trust policy must allow the immediately previous role, not the original caller."
  resolution: "Update the trust policy to allow the ARN of the role that is directly assuming it."

## Output Format

```yaml
root_cause: "chained_roles — <specific_cause>"
evidence:
  - type: role_chain
    content: "<Role A> → <Role B> → <Role C>"
  - type: session_duration
    content: "limited to 3600 seconds (chained)"
  - type: cloudtrail_events
    content: "<sequential AssumeRole events>"
severity: MEDIUM
mitigation:
  immediate: "Restructure to avoid role chaining or accept 1-hour limit"
  long_term: "Use AWS SSO/Identity Center or direct role assumption"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws cloudtrail lookup-events` (AssumeRole) | GREEN | Read-only diagnostic |
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws iam get-role` | GREEN | Read-only diagnostic |
| `aws iam update-assume-role-policy` (restructure chain) | RED | Trust policy changes affect which principals can assume roles in the chain |
| `aws sts assume-role --source-identity` | YELLOW | Creates a session with source identity tracking |

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

- No temporary resources are created during chained role analysis

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
