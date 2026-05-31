---
title: "E1 — SCP Blocking Operations"
description: "Diagnose SCP deny rules blocking operations in member accounts"
status: active
severity: HIGH
triggers:
  - "SCP"
  - "service control policy"
  - "organization policy"
  - "SCP blocking"
  - "SCP deny"
owner: devops-agent
objective: "Identify which SCP is blocking the operation and determine the correct remediation"
context: "SCPs restrict the maximum permissions available to principals in member accounts. SCPs do NOT apply to the management account or service-linked roles. SCPs use an allow-list or deny-list model. If an SCP doesn't allow an action, it's implicitly denied even if IAM policies allow it."
---

## Phase 1 — Triage

MUST:
- Verify the account is a member account (not management): `aws organizations describe-account --account-id <account-id>`
- List SCPs applied to the account: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- List SCPs applied to the OU: `aws organizations list-parents --child-id <account-id>` then `aws organizations list-policies-for-target --target-id <ou-id> --filter SERVICE_CONTROL_POLICY`
- Get each SCP document: `aws organizations describe-policy --policy-id <policy-id>`

SHOULD:
- Check all OU levels — SCPs are inherited from parent OUs to child OUs to accounts
- Verify the FullAWSAccess SCP is still attached (removing it denies everything)
- Check CloudTrail in the member account for the denied event

MAY:
- Check if the action is performed by a service-linked role (SCPs don't apply to SLRs)
- Use the Organizations console for a visual SCP inheritance view

## Phase 2 — Remediate

MUST:
- If using allow-list model: add the required action to the SCP Allow statement
- If using deny-list model: remove the action from the SCP Deny statement or add an exception condition
- Apply changes at the correct OU level: `aws organizations update-policy --policy-id <id> --content file://scp.json`

SHOULD:
- Add conditions to SCP Deny statements to allow exceptions (e.g., specific roles)
- Test SCP changes in a sandbox OU before applying to production
- Verify the fix doesn't open unintended access in other accounts

MAY:
- Use SCP simulation tools to test policy changes before applying
- Implement SCP changes through IaC (CloudFormation StackSets, Terraform)

## Common Issues

- symptoms: "Action works in management account but fails in member account"
  diagnosis: "SCPs only apply to member accounts. The management account is exempt."
  resolution: "Check SCPs on the member account. Add the action to the SCP Allow or remove from Deny."

- symptoms: "SCP allows the action but it's still denied"
  diagnosis: "SCPs are inherited. A parent OU's SCP may be denying the action."
  resolution: "Check SCPs at all OU levels from root to the account."

- symptoms: "Service-linked role works but regular role is denied"
  diagnosis: "SCPs don't apply to service-linked roles. The SCP is blocking the regular role."
  resolution: "This is expected behavior. Add the action to the SCP for regular roles if needed."

## Output Format

```yaml
root_cause: "scp_deny — <scp_name>: <denied_action>"
evidence:
  - type: scp_document
    content: "<relevant SCP statement>"
  - type: scp_inheritance
    content: "Root OU → <OU path> → Account"
  - type: cloudtrail_event
    content: "<denied event>"
severity: HIGH
mitigation:
  immediate: "Modify SCP to allow the required action"
  long_term: "Implement SCP testing in sandbox OUs before production deployment"
```


## Safety Ratings

safety_ratings:
  - "SCP modification (changes affect all principals in the target OU or account): RED — SCP changes affect the maximum permissions for every principal in every account under the target OU, potentially breaking workloads across the entire organizational unit"

## Escalation Conditions

escalation_conditions:
  - "ALWAYS escalate to a human before modifying any SCP"
  - "Escalate if the SCP is attached at the root OU level"
  - "Escalate if the change could affect multiple accounts or OUs"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Reattach the original SCP using `aws organizations update-policy --policy-id <id> --content file://original-scp.json`, or if the SCP was detached, reattach it using `aws organizations attach-policy --policy-id <id> --target-id <ou-or-account-id>`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws organizations describe-policy` returns full SCP documents revealing organizational security controls — classification: HIGH
- `aws organizations list-policies-for-target` reveals the SCP inheritance chain — classification: HIGH
- `aws cloudtrail lookup-events` may reveal denied operations and caller details — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during SCP troubleshooting

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
