---
title: "D3 — Account Drift"
description: "Diagnose and resolve drift when accounts are moved or modified outside Control Tower"
status: active
severity: HIGH
triggers:
  - "Account moved"
  - "Account drift"
  - "Account not governed"
  - "Account in wrong OU"
owner: devops-agent
objective: "Identify account-level drift and restore governance"
context: "Account drift occurs when a governed account is moved to a different OU outside of Control Tower, or when CT-managed resources in the account are modified. Moving accounts changes which controls apply."
---

## Phase 1 — Triage

MUST:
- Identify the drifted account and its current OU
- Check if the account was moved: `aws organizations list-parents --child-id <account-id>`
- Verify CT-managed resources in the account (AWSControlTowerExecution role, Config recorder, CT stacks)
- Check if the account's current OU is registered with CT

SHOULD:
- Check CloudTrail for MoveAccount API calls
- Verify the account's baseline StackSet instances are healthy
- Check if controls from the previous OU still apply

MAY:
- Compare the account's current governance state with expected state
- Check if the account was moved to an unregistered OU

## Phase 2 — Remediate

MUST:
- Move the account back to the correct registered OU, OR
- Update the account in CT to reflect the new OU (re-register the target OU)
- Verify controls are correctly applied after the move

SHOULD:
- Re-register both the source and target OUs to ensure clean state
- Verify StackSet instances are updated for the account
- Check compliance status after remediation

MAY:
- Implement account move notifications via EventBridge
- Restrict MoveAccount API via IAM policies

## Common Issues

- symptoms: "Account shows as drifted in CT console"
  diagnosis: "Account was moved to a different OU outside of CT."
  resolution: "Move the account back or update CT by re-registering the OUs."

- symptoms: "Account not receiving StackSet updates"
  diagnosis: "Account is in an unregistered OU or StackSet targeting is broken."
  resolution: "Move to a registered OU and re-register the OU."

## Safety Ratings

```
safety_ratings:
  - "list-parents: GREEN — read-only account parent lookup"
  - "describe-account: GREEN — read-only account metadata"
  - "re-register source and target OUs: YELLOW — recoverable, resets CT baselines"
  - "move account back to correct OU: YELLOW — recoverable, account can be moved again"
  - "move account between OUs in production: RED — changes which controls apply, can break governance"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Fix requires moving the Audit or Log Archive account
- Drift affects multiple OUs or accounts
- Account was moved to an unregistered OU
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Account governance state and StackSet instance details

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If moving an account back to its original OU causes issues, verify the OU is still registered and controls are enabled
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the account's current OU and target OU before making any moves
- If the account was moved to an unregistered OU, register the OU first before moving the account back — do NOT leave accounts in unregistered OUs

## Output Format

```yaml
root_cause: "account_drift — <account_id> moved from <source_ou> to <target_ou>"
evidence:
  - type: organizations_api
    content: "<account parent OU>"
  - type: cloudtrail_events
    content: "<MoveAccount event>"
severity: HIGH
mitigation:
  immediate: "Move account to correct OU or re-register OUs"
  long_term: "Implement account move monitoring and restrictions"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
