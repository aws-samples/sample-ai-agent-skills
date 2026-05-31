---
title: "C1 — Performance Counters Issues"
description: "Diagnose Fleet Manager performance counters display and data issues"
status: active
severity: MEDIUM
triggers: ["performance counters not showing", "counters not loading", "no performance data"]
owner: devops-agent
objective: "Resolve Fleet Manager performance counters display issues"
context: "Performance counters are Windows-only. They display Windows Performance Monitor data through Fleet Manager. Requires SSM Agent 3.0+ and Windows managed nodes."
---

## Phase 1 — Triage

MUST:
- Verify instance is Windows: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].{Id:InstanceId,Platform:PlatformType,Agent:AgentVersion}'`
- Confirm SSM Agent version is 3.0+
- Check connection status: `aws ssm get-connection-status --target <instance-id>`

SHOULD:
- Verify Windows Performance Monitor service is running on the instance
- Check if the instance has sufficient resources (CPU/memory) to collect counters

MAY:
- Test Session Manager connectivity independently
- Review Windows Event Logs for Performance Monitor errors

## Phase 2 — Remediate

MUST:
- Confirm the instance is Windows — performance counters are NOT available for Linux
- Update SSM Agent to 3.0+ if outdated
- Ensure Windows Performance Monitor service (PerfMon) is running

SHOULD:
- Restart SSM Agent on the instance: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-RunPowerShellScript --parameters commands=["Restart-Service AmazonSSMAgent"]`
- Verify WMI (Windows Management Instrumentation) service is running

MAY:
- Rebuild WMI repository if corrupted: `winmgmt /salvagerepository`

## Common Issues

- symptoms: "Performance counters tab shows 'Not available' for a Linux instance"
  diagnosis: "Performance counters are Windows-only feature."
  resolution: "Use CloudWatch Agent for Linux monitoring. Performance counters are not supported on Linux."

- symptoms: "Performance counters tab loads but shows no data on Windows"
  diagnosis: "Windows Performance Monitor service stopped or WMI corrupted."
  resolution: "Restart PerfMon and WMI services. Rebuild WMI repository if needed."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only platform check"
  - "get-connection-status: GREEN — read-only connectivity check"
  - "send-command restart agent: YELLOW — restarts service on instance"
  - "WMI repository rebuild: RED — modifies Windows system component"
```

## Escalation Conditions

- Performance counters fail on all Windows instances simultaneously
- WMI repository is corrupted and rebuild does not resolve the issue
- Performance counter collection causes high CPU usage on the instance
- Custom performance counters required for compliance are not displaying

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- Performance counter data: LOW — system metrics (CPU, memory, disk, network)
- `aws ssm send-command`: MEDIUM — executes commands on the instance
- Windows Event Logs: MEDIUM — may contain system and application error details

## Prohibited Actions

- NEVER suggest performance counters for Linux instances
- NEVER disable Windows Performance Monitor permanently
- NEVER rebuild WMI repository on production instances without backup
- NEVER install third-party performance monitoring tools without security review

## Phase 3 — Rollback

1. If WMI repository was rebuilt and caused issues, restore from system backup
2. If SSM Agent was updated and counters broke, downgrade to previous version
3. If Windows services were modified, restore to their previous startup configuration
4. Verify performance counters display correctly after remediation
5. Document any Windows system changes made during troubleshooting

## Output Format

```yaml
root_cause: "performance_counters — <specific_cause>"
evidence:
  - type: counter_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Restore performance counter display"
  long_term: "Monitor Windows service health with CloudWatch"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
