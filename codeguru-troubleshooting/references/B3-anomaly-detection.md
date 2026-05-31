---
title: "B3 — Anomaly Detection Issues"
description: "Diagnose profiler anomaly detection and findings"
status: active
severity: LOW
triggers:
  - "no anomalies detected"
  - "false anomaly"
  - "anomaly detection not working"
owner: devops-agent
objective: "Resolve anomaly detection issues"
context: "Anomaly detection requires baseline data. New profiling groups need time."
---

## Phase 1 — Triage

MUST:
- Check findings: `aws codeguru-profiler list-findings-reports --profiling-group-name <name> --start-time <start> --end-time <end>`
- Verify profiling group has sufficient baseline data
- Check anomaly detection configuration

SHOULD:
- Verify profiling group age (needs days of data)
- Check for notification configuration

## Phase 2 — Remediate

MUST:
- Wait for baseline to establish (days of data)
- Verify agent is collecting data consistently

SHOULD:
- Configure anomaly notifications
- Review findings regularly

## Common Issues

- symptoms: "No anomalies detected on new profiling group"
  diagnosis: "Insufficient baseline data."
  resolution: "Wait for several days of profiling data to establish baseline."

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: findings
    content: "<anomaly detection status>"
severity: LOW
mitigation:
  immediate: "Wait for baseline"
  long_term: "Configure anomaly notifications"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Wait for baseline to establish | GREEN |
| Verify agent is collecting data consistently | GREEN |
| Configure anomaly notifications | GREEN |
| Review findings regularly | GREEN |

## Escalation Conditions

- Fix requires persistent profiling data collection issues
- Anomaly detection not functioning after sufficient baseline period

## Data Sensitivity

- **MEDIUM**: anomaly detection findings reveal performance regressions and resource usage patterns
- Baseline data contains historical application behavior profiles

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If notification configuration was changed, restore previous settings
2. If profiling group configuration was modified, restore previous settings
3. No rollback needed for waiting on baseline data
4. Verify rollback by confirming anomaly detection configuration matches expected state

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
