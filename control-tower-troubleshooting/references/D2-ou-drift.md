---
title: "D2 — OU Drift"
description: "Diagnose and resolve drift detected at the organizational unit level"
status: active
severity: HIGH
triggers:
  - "OU drift"
  - "OU deleted"
  - "OU renamed"
  - "Registered OU modified"
owner: devops-agent
objective: "Identify OU-level drift and restore the expected state"
context: "OU drift occurs when a registered OU is renamed, deleted, or has its CT-managed SCPs modified. This breaks control enforcement and StackSet targeting for all accounts in the OU."
---

## Phase 1 — Triage

MUST:
- List registered OUs and check their status
- Verify OU names match CT expectations
- Check SCPs attached to the OU: `aws organizations list-policies-for-target --target-id <ou-id> --filter SERVICE_CONTROL_POLICY`
- Check if any registered OU was deleted from Organizations

SHOULD:
- Check CloudTrail for DeleteOrganizationalUnit, UpdateOrganizationalUnit, or DetachPolicy events
- Verify StackSet targeting still includes the OU
- Check if accounts in the OU are still governed

MAY:
- Compare current OU structure with expected CT structure
- Check if new unregistered OUs were created

## Phase 2 — Remediate

MUST:
- If OU renamed: re-register the OU to update CT's records
- If OU deleted: recreate the OU and re-register it, then move accounts back
- If SCP detached: re-register the OU to reattach CT-managed SCPs

SHOULD:
- Re-register the OU to reset all CT-managed resources
- Verify controls are re-enabled after re-registration
- Check account governance status after OU fix

MAY:
- Implement OU modification prevention via SCP on the management account (limited effectiveness since SCPs don't apply to management account)
- Set up CloudWatch Events for OU modification API calls

## Common Issues

- symptoms: "Controls show as not enabled after OU change"
  diagnosis: "OU was renamed or recreated, breaking CT's OU tracking."
  resolution: "Re-register the OU with Control Tower."

- symptoms: "StackSets not deploying to accounts in the OU"
  diagnosis: "OU drift broke StackSet targeting."
  resolution: "Re-register the OU to restore StackSet deployment targets."

## Safety Ratings

```
safety_ratings:
  - "list-organizational-units-for-parent: GREEN — read-only OU listing"
  - "list-policies-for-target: GREEN — read-only SCP listing"
  - "describe-organizational-unit: GREEN — read-only OU metadata"
  - "re-register OU: YELLOW — recoverable, resets CT baselines for the OU"
  - "recreate deleted OU and move accounts back: RED — can break governance if accounts are moved between OUs incorrectly"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Registered OU was deleted from Organizations
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudTrail OU modification events

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If a deleted OU is recreated, verify all accounts are moved back to the correct OU before re-registering
- Document the OU structure and account membership before making any changes
- If OU rename caused drift, re-registration will fix it — do NOT try to rename the OU back manually

## Output Format

```yaml
root_cause: "ou_drift — <ou_name> <modification_type>"
evidence:
  - type: organizations_api
    content: "<OU state details>"
  - type: cloudtrail_events
    content: "<OU modification events>"
severity: HIGH
mitigation:
  immediate: "Re-register the OU to restore CT governance"
  long_term: "Monitor OU modifications and restrict direct Organizations changes"
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
