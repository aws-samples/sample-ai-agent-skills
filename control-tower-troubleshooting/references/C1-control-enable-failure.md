---
title: "C1 — Control Enable Failure"
description: "Diagnose failures when enabling controls (guardrails) on an OU"
status: active
severity: HIGH
triggers:
  - "EnableControl failed"
  - "Cannot enable guardrail"
  - "Control enable error"
owner: devops-agent
objective: "Identify why the control failed to enable and enable it successfully"
context: "Controls are enabled per OU. Preventive controls deploy SCPs, detective controls deploy Config rules via StackSets, and proactive controls deploy CloudFormation hooks. Each type has different failure modes."
---

## Phase 1 — Triage

MUST:
- Check the control operation status: `aws controltower list-enabled-controls --target-identifier <ou-arn>`
- Identify the control type (preventive, detective, proactive)
- Check for drift on the target OU
- For detective controls: check StackSet deployment status for Config rule stacks
- For preventive controls: check SCP attachment status

SHOULD:
- Verify the landing zone version supports the control
- Check if the control has dependencies on other controls
- Verify the OU is registered with Control Tower

MAY:
- Check CloudTrail for the EnableControl API call and error
- Review the control's documentation for prerequisites

## Phase 2 — Remediate

MUST:
- Resolve any drift on the OU before retrying
- For detective controls: fix failed StackSet instances in member accounts
- For preventive controls: check SCP limits (max 5 SCPs per target)
- Ensure no concurrent control operations are running

SHOULD:
- Verify the OU has accounts (some controls require at least one account)
- Check if the control conflicts with existing custom SCPs
- Re-register the OU if the control infrastructure is corrupted

MAY:
- Enable controls in a specific order if dependencies exist
- Test the control on a sandbox OU first

## Common Issues

- symptoms: "Detective control enable fails with StackSet error"
  diagnosis: "Config rule StackSet failed to deploy to one or more member accounts."
  resolution: "Check failed stack instances. Common causes: missing AWSControlTowerExecution role, Config recorder issues."

- symptoms: "Preventive control fails with SCP limit exceeded"
  diagnosis: "The OU or account has reached the maximum number of SCPs (5)."
  resolution: "Consolidate existing SCPs or remove unused ones to make room."

- symptoms: "Control not available for the OU"
  diagnosis: "The control requires a newer landing zone version or is not supported for the OU type."
  resolution: "Update the landing zone or check control compatibility."

## Safety Ratings

```
safety_ratings:
  - "list-enabled-controls: GREEN — read-only control listing"
  - "list-stack-instances: GREEN — read-only StackSet instance listing"
  - "list-policies-for-target: GREEN — read-only SCP listing"
  - "enable/disable control: YELLOW — recoverable, control can be toggled back"
  - "re-register OU to fix control infrastructure: YELLOW — recoverable, resets CT baselines"
  - "consolidate SCPs to make room: RED — modifying SCPs can break governance for all accounts in the OU"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- SCP limit (5) reached and consolidation requires modifying existing SCPs
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet deployment details (account IDs, regions)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If enabling a control causes workload disruption, disable the control immediately — this is recoverable
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document which controls were enabled/disabled and on which OUs before making changes
- If SCP consolidation was performed, keep a backup of the original SCP documents

## Output Format

```yaml
root_cause: "control_enable_failure — <control_id> on <ou>"
evidence:
  - type: controltower_api
    content: "<enable control error details>"
  - type: stackset_status
    content: "<StackSet deployment status>"
severity: HIGH
mitigation:
  immediate: "Fix the blocking condition and retry enable"
  long_term: "Monitor SCP limits and StackSet health proactively"
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
