---
title: "D2 — Bandwidth Throttling Issues"
description: "Diagnose bandwidth throttling configuration and impact on transfers"
status: active
severity: MEDIUM
triggers:
  - "bandwidth throttling"
  - "BytesPerSecond"
  - "rate limiting"
  - "throttle configuration"
owner: devops-agent
objective: "Identify and resolve bandwidth throttling configuration issues"
context: "DataSync bandwidth throttling (BytesPerSecond) limits per-task execution throughput. It does not apply per-agent or per-account. A value of -1 means no throttling. Throttling is useful to avoid saturating network links but can cause unnecessarily slow transfers if misconfigured."
---

## Phase 1 — Triage

MUST:
- Check task throttling configuration: `aws datasync describe-task --task-arn <task-arn> --query 'Options.BytesPerSecond'`
- Check current execution throughput: `aws cloudwatch get-metric-statistics --namespace AWS/DataSync --metric-name BytesTransferred --dimensions Name=TaskId,Value=<task-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Verify if throttling is intentional or accidental

SHOULD:
- Check if multiple tasks are running concurrently on the same agent
- Compare throttle value with available network bandwidth
- Review task execution history for throughput patterns

MAY:
- Calculate optimal throttle value based on network capacity and other traffic
- Check if network QoS policies are also throttling DataSync traffic

## Phase 2 — Remediate

MUST:
- Adjust BytesPerSecond to appropriate value or remove throttling (-1):
  `aws datasync update-task --task-arn <task-arn> --options BytesPerSecond=-1`
- Or set specific limit: `aws datasync update-task --task-arn <task-arn> --options BytesPerSecond=125000000` (1 Gbps)

SHOULD:
- Coordinate throttling across concurrent tasks sharing the same network path
- Schedule bandwidth-intensive tasks during off-peak hours

MAY:
- Implement network-level QoS instead of per-task throttling
- Use AWS Direct Connect with dedicated bandwidth for DataSync

## Common Issues

- symptoms: "Transfer capped at specific speed despite available bandwidth"
  diagnosis: "BytesPerSecond is set to a value lower than available bandwidth."
  resolution: "Increase BytesPerSecond or set to -1 for no throttling."

- symptoms: "Network saturated by DataSync affecting other traffic"
  diagnosis: "No throttling configured and DataSync uses all available bandwidth."
  resolution: "Set BytesPerSecond to limit DataSync bandwidth usage."

- symptoms: "Throttle setting not taking effect"
  diagnosis: "Throttle applies to new executions only, not running ones."
  resolution: "Cancel current execution and start a new one with updated settings."

## Output Format

```yaml
root_cause: "bandwidth_throttling — <specific_cause>"
evidence:
  - type: throttle_config
    content: "<BytesPerSecond value>"
  - type: actual_throughput
    content: "<measured transfer throughput>"
  - type: network_capacity
    content: "<available network bandwidth>"
severity: MEDIUM
mitigation:
  immediate: "Adjust BytesPerSecond to appropriate value"
  long_term: "Implement bandwidth management strategy for DataSync tasks"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Throttling configuration is operational metadata |
| Blast Radius | MEDIUM — Removing throttling can saturate shared network links |
| Rollback Complexity | LOW — Throttle changes apply to new executions only |
| Compliance Scope | LOW — Throttling is an operational setting |

## Escalation Conditions

- Escalate to network team before removing or increasing bandwidth throttling
- Escalate to operations if throttling causes SLA-critical transfers to miss deadlines

## Data Sensitivity

- **Throttle configuration** is operational metadata
- **Network bandwidth data** reveals infrastructure capacity

## PROHIBITED Actions

- Do NOT remove bandwidth throttling on shared network links without network team approval
- Do NOT set throttling to -1 (unlimited) during business hours on shared links
- Do NOT change throttling on running executions — it only applies to new ones

## Phase 3 — Rollback

- Restore previous BytesPerSecond value: `aws datasync update-task --task-arn <arn> --options BytesPerSecond=<previous-value>`
- Cancel current execution and start new one for throttle changes to take effect

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
