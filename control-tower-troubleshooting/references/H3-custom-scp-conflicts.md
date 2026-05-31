---
title: "H3 — Custom SCP Conflicts"
description: "Diagnose conflicts between custom SCPs and Control Tower operations"
status: active
severity: HIGH
triggers:
  - "Custom SCP blocking CT"
  - "SCP conflict with Control Tower"
  - "CT operation fails due to SCP"
owner: devops-agent
objective: "Resolve conflicts between custom SCPs and CT-managed operations"
context: "Custom SCPs can inadvertently block Control Tower operations like StackSet deployments, Config recorder setup, or CloudTrail configuration. CT uses specific IAM roles that must not be restricted by custom SCPs."
---

## Phase 1 — Triage

MUST:
- Identify the CT operation that's failing
- List all custom SCPs on the affected OU/account: `aws organizations list-policies-for-target --target-id <id> --filter SERVICE_CONTROL_POLICY`
- Check if custom SCPs deny actions needed by CT (CloudFormation, Config, CloudTrail, IAM)
- Verify the AWSControlTowerExecution role is not blocked by SCPs

SHOULD:
- Check if the custom SCP has conditions that exclude CT roles
- Review the SCP deny statements for overly broad patterns
- Check if the SCP was recently added or modified

MAY:
- Use IAM policy simulator to test effective permissions
- Compare the SCP with CT's required permissions

## Phase 2 — Remediate

MUST:
- Add condition keys to custom SCPs to exclude CT roles:
  ```json
  "Condition": {
    "ArnNotLike": {
      "aws:PrincipalARN": "arn:aws:iam::*:role/AWSControlTower*"
    }
  }
  ```
- Verify CT operations succeed after SCP modification

SHOULD:
- Test SCP changes in a sandbox OU first
- Document CT role exceptions in SCP documentation
- Review all custom SCPs for potential CT conflicts

MAY:
- Implement SCP validation that checks for CT compatibility
- Create SCP templates that include CT exceptions by default

## Common Issues

- symptoms: "StackSet deployment fails in accounts with custom SCPs"
  diagnosis: "Custom SCP blocks CloudFormation or IAM actions needed by CT StackSets."
  resolution: "Add condition to exclude AWSControlTower* roles from the SCP deny."

- symptoms: "Account enrollment fails with access denied"
  diagnosis: "Custom SCP blocks the AWSControlTowerExecution role."
  resolution: "Exclude the AWSControlTowerExecution role from the SCP deny statements."

- symptoms: "Config recorder keeps stopping"
  diagnosis: "Custom SCP blocks Config API calls."
  resolution: "Exclude CT roles from the SCP or add Config actions to the allow list."

## Safety Ratings

```
safety_ratings:
  - "list-policies-for-target: GREEN — read-only SCP listing"
  - "describe-policy: GREEN — read-only SCP content"
  - "add condition keys to custom SCP: YELLOW — recoverable, SCP can be reverted"
  - "modify custom SCP deny statements: RED — can weaken security boundaries or break workloads"
  - "remove custom SCP: RED — removes security boundary for all accounts in the target"
```

## Escalation Conditions

- Fix requires modifying CT-managed SCPs
- Fix requires updating the landing zone version
- Custom SCP blocks CT operations across multiple OUs
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Custom SCP was deployed via CfCT and requires pipeline changes

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: IAM role ARN patterns (expose CT role naming conventions)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If custom SCP modification breaks workloads, revert the SCP to its previous version immediately
- Keep a backup of all SCP content before modifications
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Test SCP changes in a sandbox OU before applying to production
- If CT role exceptions were added to a custom SCP, verify the exceptions don't weaken other security controls

## Output Format

```yaml
root_cause: "custom_scp_conflict — <scp_name> blocking <ct_operation>"
evidence:
  - type: scp_policy
    content: "<conflicting deny statement>"
severity: HIGH
mitigation:
  immediate: "Add CT role exceptions to the custom SCP"
  long_term: "Include CT exceptions in all custom SCP templates"
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
