---
title: "C1 — Run Command Failures"
description: "Diagnose Run Command execution failures"
status: active
severity: HIGH
triggers:
  - "command failed"
  - "Run Command error"
  - "SendCommand failed"
  - "command status Failed"
owner: devops-agent
objective: "Identify why Run Command executions are failing and fix the issue"
context: "Run Command sends commands to managed instances via the ec2messages endpoint. Failures can be due to agent connectivity, IAM permissions, document errors, or instance-level execution issues. Command output is available via get-command-invocation."
---

## Phase 1 — Triage

MUST:
- List failed commands: `aws ssm list-commands --filters key=Status,value=Failed --max-results 10`
- Get command details: `aws ssm list-command-invocations --command-id <cmd-id> --details`
- Get specific invocation output: `aws ssm get-command-invocation --command-id <cmd-id> --instance-id <instance-id>`
- Check instance connectivity: `aws ssm get-connection-status --target <instance-id>`

SHOULD:
- Check the StandardErrorContent in the invocation output
- Verify the document exists and is valid: `aws ssm describe-document --name <document-name>`
- Check CloudTrail for SendCommand events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=SendCommand --max-results 10`

MAY:
- Check if the command was cancelled: `aws ssm list-commands --command-id <cmd-id>`
- Verify instance has ec2messages endpoint access
- Check SSM Agent logs on the instance

## Phase 2 — Remediate

MUST:
- Fix the identified error (permissions, script errors, connectivity)
- Ensure target instances are managed and connected
- Verify the SSM document is correct and compatible

SHOULD:
- Use proper error handling in scripts
- Set appropriate timeout values for long-running commands
- Use output S3 bucket for large command outputs (>48,000 characters truncated)

MAY:
- Implement command execution monitoring with CloudWatch Events
- Use rate control for large-scale command execution

## Common Issues

- symptoms: "InvocationDoesNotExist"
  diagnosis: "Instance did not receive the command. Agent not connected."
  resolution: "Verify instance is managed and has ec2messages endpoint access."

- symptoms: "AccessDeniedException on SendCommand"
  diagnosis: "Caller lacks ssm:SendCommand permission or instance not in target."
  resolution: "Grant ssm:SendCommand permission. Verify instance tags match targeting."

- symptoms: "Command output truncated"
  diagnosis: "Output exceeds 48,000 character limit in API response."
  resolution: "Configure S3 output bucket in the command parameters."

## Output Format

```yaml
root_cause: "command_failure — <specific_cause>"
evidence:
  - type: command_status
    content: "<list-commands output>"
  - type: invocation_output
    content: "<get-command-invocation output>"
severity: HIGH
mitigation:
  immediate: "Fix the identified execution error"
  long_term: "Implement command monitoring and output logging to S3"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
