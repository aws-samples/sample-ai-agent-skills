---
title: "B2 — Profiling Data Issues"
description: "Diagnose profiling data quality and visualization"
status: active
severity: MEDIUM
triggers:
  - "profiling data incomplete"
  - "flame graph empty"
  - "data gaps"
owner: devops-agent
objective: "Resolve profiling data issues"
context: "Profiling data takes 5-15 minutes to appear. Gaps indicate agent issues."
---

## Phase 1 — Triage

MUST:
- Check profiling group: `aws codeguru-profiler describe-profiling-group --profiling-group-name <name>`
- Verify agent is running (check application logs)
- Check data timeline for gaps
- Verify IAM permissions for data submission

SHOULD:
- Check agent sampling configuration
- Verify network connectivity
- Check for application restarts

## Phase 2 — Remediate

MUST:
- Restart agent if not running
- Fix network connectivity
- Fix IAM permissions

SHOULD:
- Monitor agent health
- Configure alerting for data gaps

## Common Issues

- symptoms: "Flame graph shows data for some periods but not others"
  diagnosis: "Application restarts or agent connectivity issues."
  resolution: "Check application uptime. Verify agent auto-restarts with application."

## Output Format

```yaml
root_cause: "profiling_data — <specific_cause>"
evidence:
  - type: data_timeline
    content: "<profiling data availability>"
severity: MEDIUM
mitigation:
  immediate: "Fix data collection"
  long_term: "Monitor agent health"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Restart agent if not running | YELLOW |
| Fix network connectivity | GREEN |
| Fix IAM permissions | YELLOW |
| Monitor agent health | GREEN |
| Configure alerting for data gaps | GREEN |

## Escalation Conditions

- Fix requires restarting production application to restore profiling
- Fix involves modifying production network or IAM configurations

## Data Sensitivity

- **HIGH**: profiling data reveals application performance patterns and resource usage
- Data gaps may indicate application instability or infrastructure issues

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If agent was restarted, verify application stability after restart
2. If network configuration was changed, restore previous settings
3. If IAM permissions were modified, revert to previous policy
4. Verify rollback by confirming profiling data collection resumes normally

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
