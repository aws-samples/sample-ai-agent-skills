---
title: "D1 — Run Command From Chat"
description: "Diagnose SSM Run Command execution from chat failures including permissions, instance connectivity, and command errors"
status: active
severity: HIGH
triggers: ["run command failed from chat", "SSM command error in chat", "cannot run command on instance", "chat command timeout"]
owner: devops-agent
objective: "Resolve SSM Run Command execution failures initiated from AWS Chatbot"
context: "Run Command from chat uses the channel IAM role to call ssm:SendCommand. Target instances must have SSM agent running and be registered. Guardrails must allow SSM operations."
---

## Phase 1 — Triage

MUST:
- Check channel IAM role for SSM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names ssm:SendCommand`
- Verify target instance SSM connectivity: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Check guardrails allow SSM commands

SHOULD:
- Review the command execution result: `aws ssm list-command-invocations --command-id <command-id> --details`
- Check if the instance has the required SSM document
- Verify the instance IAM role has SSM agent permissions

MAY:
- Test the command directly via AWS CLI: `aws ssm send-command --instance-ids <id> --document-name <doc>`
- Check SSM agent logs on the instance

## Phase 2 — Remediate

MUST:
- Grant the channel IAM role ssm:SendCommand permission
- Ensure target instances have SSM agent installed and running
- Update guardrails to allow SSM operations

SHOULD:
- Restrict SSM commands to specific documents and instances in guardrails
- Verify instance IAM role includes AmazonSSMManagedInstanceCore policy
- Test command execution in a non-production environment

MAY:
- Set up predefined command templates for common operations
- Implement command approval workflows for production instances

## Safety Ratings

```
safety_ratings:
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "ssm describe-instance-information: GREEN — read-only instance status"
  - "ssm send-command: RED — executes commands on instances"
  - "ssm list-command-invocations: GREEN — read-only command results"
```

## Escalation Conditions

- Command execution on production instances fails with unknown errors
- SSM agent connectivity issues affect multiple instances
- Command execution produces unexpected results on production systems
- Guardrail changes required for incident response commands

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — returns allow/deny results
- `aws ssm describe-instance-information`: MEDIUM — returns instance details
- `aws ssm send-command`: HIGH — executes commands on instances
- `aws ssm list-command-invocations`: HIGH — contains command output

## Prohibited Actions

- NEVER run destructive commands on production instances from chat without approval
- NEVER grant ssm:SendCommand with wildcard instance targets
- NEVER bypass guardrails for SSM commands
- NEVER share command output containing sensitive data in public channels

## Phase 3 — Rollback

1. If a command made unwanted changes, connect to the instance and revert
2. If IAM permissions were over-granted, revert to previous policy
3. If guardrails were loosened, restore previous restrictions
4. Verify instance state after command execution
5. Document the command execution and any rollback actions

## Common Issues

- symptoms: "Run command returns 'access denied' in chat"
  diagnosis: "The channel IAM role lacks ssm:SendCommand permission, or guardrails block SSM operations."
  resolution: "Add ssm:SendCommand to the channel role. Update guardrails to allow SSM actions."

- symptoms: "Command sent but no response received"
  diagnosis: "The target instance SSM agent is not running or the instance is not registered with SSM."
  resolution: "Verify SSM agent status. Check instance IAM role. Ensure the instance can reach SSM endpoints."

- symptoms: "Command fails with 'document not found'"
  diagnosis: "The SSM document specified in the command does not exist in the target region."
  resolution: "Verify the document name and region. Use AWS-RunShellScript or AWS-RunPowerShellScript for ad-hoc commands."

## Output Format

```yaml
root_cause: "run_command_chat — <specific_cause>"
evidence:
  - type: command_error
    content: "<command execution failure details>"
severity: HIGH
mitigation:
  immediate: "Fix command execution issue"
  long_term: "Set up predefined command templates with guardrails"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
