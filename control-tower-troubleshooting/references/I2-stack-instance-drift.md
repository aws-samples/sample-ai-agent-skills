---
title: "I2 — Stack Instance Drift"
description: "Diagnose drift in Control Tower-managed CloudFormation stack instances"
status: active
severity: MEDIUM
triggers:
  - "Stack instance drifted"
  - "Stack drift detected"
  - "CT resources modified in account"
owner: devops-agent
objective: "Detect and resolve drift in CT-managed stack instances"
context: "Stack instance drift occurs when resources deployed by CT StackSets are modified directly in member accounts. This can break governance, disable Config recorders, or modify IAM roles."
---

## Phase 1 — Triage

MUST:
- Detect drift on CT stack instances: `aws cloudformation detect-stack-set-drift --stack-set-name <name>`
- Check drift status: `aws cloudformation describe-stack-set-operation --stack-set-name <name> --operation-id <id>`
- Identify which resources drifted and how
- Check CloudTrail in the affected account for the modification

SHOULD:
- Assess the impact of the drift (is governance broken?)
- Check if the drift affects Config, CloudTrail, or IAM roles
- Determine if the drift was intentional or accidental

MAY:
- Compare the actual resource configuration with the expected template
- Check if multiple accounts have similar drift

## Phase 2 — Remediate

MUST:
- Re-register the OU to reset stack instances to expected state
- Verify the drifted resources are restored after re-registration

SHOULD:
- Identify and address the root cause of the drift
- Implement preventive measures (SCPs to block modifications)
- Document the drift incident

MAY:
- Set up periodic drift detection
- Implement CloudWatch Events for resource modification alerts

## Common Issues

- symptoms: "Config recorder configuration changed"
  diagnosis: "Someone modified the Config recorder in a member account."
  resolution: "Re-register the OU to restore the Config recorder configuration."

- symptoms: "CT IAM role permissions modified"
  diagnosis: "Someone changed the AWSControlTowerExecution role policy."
  resolution: "Re-register the OU to restore the role. Implement SCP to prevent role modification."

## Safety Ratings

```
safety_ratings:
  - "detect-stack-set-drift: GREEN — read-only drift detection"
  - "describe-stack-set-operation: GREEN — read-only operation status"
  - "re-register OU to reset stack instances: YELLOW — recoverable, resets CT baselines"
  - "implement SCPs to block resource modifications: RED — can break workloads if too broad"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects CT-managed Config recorders or CloudTrail configuration
- Drift affects multiple OUs or accounts
- CT IAM role permissions were modified
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Drifted resource details (actual vs expected configurations)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the drifted resources and their actual configuration before re-registration restores the expected state
- If the drift was intentional (e.g., Config recorder customization), the change will be overwritten by re-registration — plan accordingly
- If SCPs were implemented to prevent future drift, test them in a sandbox OU first

## Output Format

```yaml
root_cause: "stack_instance_drift — <resource> in <account>"
evidence:
  - type: drift_detection
    content: "<drifted resources and changes>"
severity: MEDIUM
mitigation:
  immediate: "Re-register OU to restore expected state"
  long_term: "Implement drift prevention and periodic detection"
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
