---
title: "C3 — Run Command Targeting Issues"
description: "Diagnose Run Command instance targeting problems"
status: active
severity: MEDIUM
triggers:
  - "command not reaching instances"
  - "targeting error"
  - "no instances found"
  - "tag targeting not working"
owner: devops-agent
objective: "Identify why Run Command is not targeting the expected instances"
context: "Run Command supports targeting by instance IDs, tags, resource groups, and all managed instances. Targeting issues arise from incorrect tag values, instances not being managed, or resource group membership problems."
---

## Phase 1 — Triage

MUST:
- Check which instances were targeted: `aws ssm list-command-invocations --command-id <cmd-id> --query 'CommandInvocations[*].{Instance:InstanceId,Status:Status}'`
- Verify instances are managed: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus}'`
- Check instance tags match targeting criteria: `aws ec2 describe-tags --filters Name=resource-id,Values=<instance-id>`
- Verify the targeting parameters used in the command

SHOULD:
- List instances matching tag filter: `aws ssm describe-instance-information --filters Key=tag:<TagKey>,Values=<TagValue>`
- Check resource group membership if using resource group targeting: `aws resource-groups list-group-resources --group-name <group-name>`
- Verify no rate control settings are limiting execution

MAY:
- Check for recently launched instances that may not yet be registered
- Verify tag propagation from Auto Scaling groups

## Phase 2 — Remediate

MUST:
- Correct targeting parameters (tags, instance IDs, resource groups)
- Ensure target instances are managed and connected
- Verify tag values match exactly (case-sensitive)

SHOULD:
- Use tag-based targeting for dynamic instance groups
- Test targeting with a dry run or small subset first
- Document targeting conventions for the team

MAY:
- Use resource groups for complex targeting criteria
- Implement tag compliance checking

## Common Issues

- symptoms: "Command sent but 0 instances targeted"
  diagnosis: "No managed instances match the targeting criteria."
  resolution: "Verify tag values are correct and instances are managed."

- symptoms: "Some instances missing from command results"
  diagnosis: "Instances not registered as managed or tags not matching."
  resolution: "Check each missing instance's managed status and tags."

- symptoms: "Tag targeting returns unexpected instances"
  diagnosis: "Tag values are case-sensitive and may have trailing spaces."
  resolution: "Verify exact tag key and value. Tags are case-sensitive."

## Output Format

```yaml
root_cause: "targeting_issue — <specific_cause>"
evidence:
  - type: command_targets
    content: "<targeted instances list>"
  - type: instance_tags
    content: "<instance tag values>"
severity: MEDIUM
mitigation:
  immediate: "Fix targeting parameters and retry"
  long_term: "Implement tag governance and targeting documentation"
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
