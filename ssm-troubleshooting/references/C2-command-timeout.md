---
title: "C2 — Command Timeout"
description: "Diagnose Run Command timeout issues"
status: active
severity: MEDIUM
triggers:
  - "command timed out"
  - "execution timeout"
  - "DeliveryTimedOut"
  - "command still running"
owner: devops-agent
objective: "Identify why commands are timing out and configure appropriate timeouts"
context: "Run Command has two timeout mechanisms: execution timeout (default 3600s, max 172800s) and idle timeout (2 hours with no output). Commands that exceed either timeout are terminated. DeliveryTimedOut means the agent never acknowledged the command."
---

## Phase 1 — Triage

MUST:
- Check command status and timing: `aws ssm list-commands --command-id <cmd-id> --query 'Commands[*].{Status:Status,Requested:RequestedDateTime,Timeout:TimeoutSeconds}'`
- Get invocation details: `aws ssm get-command-invocation --command-id <cmd-id> --instance-id <instance-id>`
- Check if status is DeliveryTimedOut vs TimedOut vs Failed
- Verify instance connectivity: `aws ssm get-connection-status --target <instance-id>`

SHOULD:
- Check the configured timeout value in the command
- Verify the script execution time expectations
- Check if the command produces periodic output

MAY:
- Review SSM Agent logs for timeout details
- Check instance CPU/memory during command execution: `aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name CPUUtilization --dimensions Name=InstanceId,Value=<instance-id> --start-time <time> --end-time <time> --period 300 --statistics Average`

## Phase 2 — Remediate

MUST:
- Set appropriate execution timeout for the command duration
- Ensure scripts produce periodic output to avoid idle timeout
- Fix connectivity issues if DeliveryTimedOut

SHOULD:
- Add progress output (echo statements) to long-running scripts
- Use Automation documents for complex multi-step workflows
- Configure S3 output bucket for capturing full output

MAY:
- Break long commands into smaller steps
- Use State Manager for recurring long-running tasks

## Common Issues

- symptoms: "DeliveryTimedOut"
  diagnosis: "Agent never received the command. Instance not connected."
  resolution: "Verify instance connectivity and ec2messages endpoint access."

- symptoms: "TimedOut after 3600 seconds"
  diagnosis: "Command exceeded default 1-hour timeout."
  resolution: "Increase timeout with --timeout-seconds parameter (max 172800)."

- symptoms: "Command killed after 2 hours of no output"
  diagnosis: "Idle timeout triggered. Script produced no stdout/stderr for 2 hours."
  resolution: "Add periodic echo/print statements to the script."

## Output Format

```yaml
root_cause: "command_timeout — <specific_cause>"
evidence:
  - type: command_details
    content: "<command status and timeout configuration>"
  - type: invocation
    content: "<invocation status and timing>"
severity: MEDIUM
mitigation:
  immediate: "Adjust timeout or add output to script"
  long_term: "Standardize timeout configurations and script patterns"
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
