---
title: "D1 — Landing Zone Drift"
description: "Diagnose and resolve drift detected at the landing zone level"
status: active
severity: CRITICAL
triggers:
  - "Landing zone drift detected"
  - "Landing zone not in sync"
  - "Drift status"
owner: devops-agent
objective: "Identify the source of landing zone drift and restore the expected state"
context: "Landing zone drift occurs when CT-managed resources at the top level are modified outside of Control Tower. This includes changes to the Security OU, shared accounts, organization trail, or foundational SCPs. Drift blocks landing zone updates."
---

## Phase 1 — Triage

MUST:
- Check landing zone drift status: `aws controltower get-landing-zone --landing-zone-identifier <arn>`
- Check for modified foundational SCPs: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- Verify Security OU exists and contains Audit and Log Archive accounts
- Check organization trail status: `aws cloudtrail describe-trails`
- Verify CT IAM roles exist in management account

SHOULD:
- Check CloudTrail for recent Organizations, IAM, or CloudFormation API calls that modified CT resources
- Compare current SCP content with expected CT baseline SCPs
- Verify the Audit and Log Archive accounts are in the correct OU

MAY:
- Check if any CT-managed CloudFormation stacks were modified
- Review Config delivery channel and recorder status

## Phase 2 — Remediate

MUST:
- Restore modified SCPs to their original content (or use CT to reset)
- Move misplaced accounts back to the correct OU
- Recreate deleted CT IAM roles
- Use the CT console "Re-register OU" or "Repair landing zone" to reset state

SHOULD:
- Document what caused the drift to prevent recurrence
- Implement SCPs or IAM policies to prevent manual modification of CT resources
- Set up CloudWatch alarms for CT resource modifications

MAY:
- Use AWS Config rules to detect CT resource modifications
- Implement a change management process for Organization-level changes

## Common Issues

- symptoms: "Landing zone shows drift, cannot update"
  diagnosis: "CT-managed resources were modified outside of Control Tower."
  resolution: "Identify the drifted resources, restore them, and repair the landing zone."

- symptoms: "Foundational SCP was modified"
  diagnosis: "Someone edited a CT-managed SCP directly."
  resolution: "Restore the SCP to its original content. Use CT to re-register the affected OU."

- symptoms: "Shared account moved to wrong OU"
  diagnosis: "Audit or Log Archive account was moved out of the Security OU."
  resolution: "Move the account back to the Security OU."

## Safety Ratings

```
safety_ratings:
  - "get-landing-zone: GREEN — read-only landing zone status"
  - "list-policies: GREEN — read-only SCP listing"
  - "describe-trails: GREEN — read-only CloudTrail config"
  - "re-register OU to reset state: YELLOW — recoverable, resets CT baselines"
  - "repair landing zone: RED — modifies foundational governance structure across all OUs"
  - "restore modified SCPs manually: RED — can cause further drift if done incorrectly"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Fix requires moving the Audit or Log Archive account
- Drift affects multiple OUs or accounts
- Foundational SCPs were deleted (not just modified)
- CT IAM roles were deleted from the management account

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudTrail modification events (expose who changed what)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- Document the current drift state and all modified resources before attempting repair
- If SCP restoration causes workload disruption, the SCP was likely intentionally modified — investigate before forcing the original content
- If shared accounts were moved back to the Security OU, verify their baseline configuration after the move

## Output Format

```yaml
root_cause: "landing_zone_drift — <specific drifted resource>"
evidence:
  - type: controltower_status
    content: "<drift details>"
  - type: cloudtrail_events
    content: "<modification events>"
severity: CRITICAL
mitigation:
  immediate: "Restore drifted resources and repair landing zone"
  long_term: "Implement drift prevention controls and monitoring"
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
