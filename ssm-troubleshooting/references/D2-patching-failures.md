---
title: "D2 — Patching Failures"
description: "Diagnose patch installation failures on managed instances"
status: active
severity: HIGH
triggers:
  - "patching failed"
  - "patch installation error"
  - "AWS-RunPatchBaseline failed"
  - "patch scan failed"
owner: devops-agent
objective: "Identify why patch operations are failing and successfully apply patches"
context: "Patching uses the AWS-RunPatchBaseline document via Run Command or Maintenance Windows. Failures can occur during scan or install operations due to OS package manager issues, disk space, network access to patch repositories, or SSM Agent problems."
---

## Phase 1 — Triage

MUST:
- Check patch command status: `aws ssm list-commands --filters key=DocumentName,value=AWS-RunPatchBaseline key=Status,value=Failed --max-results 10`
- Get patch command output: `aws ssm get-command-invocation --command-id <cmd-id> --instance-id <instance-id>`
- Check instance patch state: `aws ssm describe-instance-patch-states --instance-ids <instance-id>`
- Verify instance connectivity: `aws ssm get-connection-status --target <instance-id>`

SHOULD:
- Check patch compliance summary: `aws ssm describe-patch-group-state --patch-group <group-name>`
- Review the StandardErrorContent for specific error messages
- Check if the instance has access to OS patch repositories

MAY:
- Check disk space on the instance via Run Command
- Verify no conflicting package manager locks
- Check maintenance window execution history: `aws ssm describe-maintenance-window-executions --window-id <window-id>`

## Phase 2 — Remediate

MUST:
- Fix the identified patching error (disk space, repo access, agent issue)
- Ensure instance can reach OS patch repositories
- Retry the patch operation after fixing the issue

SHOULD:
- Run a scan operation first to identify missing patches
- Schedule patching during maintenance windows
- Configure reboot behavior appropriately (RebootOption parameter)

MAY:
- Use patch exceptions to skip problematic patches
- Implement pre-patch and post-patch scripts via Install hooks

## Common Issues

- symptoms: "Patch scan succeeds but install fails"
  diagnosis: "Insufficient disk space or package manager conflict."
  resolution: "Free disk space. Clear package manager locks. Retry installation."

- symptoms: "Cannot access patch repository"
  diagnosis: "Instance cannot reach OS vendor repositories (no internet or proxy)."
  resolution: "Configure NAT Gateway, proxy, or local patch repository."

- symptoms: "Reboot required but not happening"
  diagnosis: "RebootOption set to NoReboot or reboot blocked by policy."
  resolution: "Set RebootOption to RebootIfNeeded. Schedule reboot in maintenance window."

## Output Format

```yaml
root_cause: "patching_failure — <specific_cause>"
evidence:
  - type: command_output
    content: "<patch command invocation output>"
  - type: patch_state
    content: "<instance patch state>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry patching"
  long_term: "Implement patch testing and staged rollout process"
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
