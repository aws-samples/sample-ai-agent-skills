---
title: "C2 — False Positives"
description: "Diagnose and reduce false positive anomaly alerts"
status: active
severity: LOW
triggers:
  - "false positive anomaly"
  - "too many anomaly alerts"
  - "anomaly noise"
  - "expected cost flagged"
owner: devops-agent
objective: "Reduce false positive anomaly detection alerts"
context: "False positives occur when expected cost changes (planned scaling, new services, seasonal patterns) trigger anomaly alerts. Tuning thresholds and providing feedback helps improve detection accuracy."
---

## Phase 1 — Triage

MUST:
- Review recent anomalies: `aws ce get-anomalies --date-interval StartDate=<start>,EndDate=<end> --max-results 20`
- Check anomaly details for root cause service and usage type
- Verify if the flagged cost change was expected (planned event)
- Check current threshold settings in subscription

SHOULD:
- Analyze anomaly frequency and patterns
- Check if specific services generate most false positives
- Review anomaly feedback history

MAY:
- Compare anomaly alerts with known change events
- Check if seasonal patterns are causing false positives

## Phase 2 — Remediate

MUST:
- Provide feedback on false positives: `aws ce provide-anomaly-feedback --anomaly-id <id> --feedback NO`
- Increase dollar threshold to filter low-impact anomalies
- Review and adjust monitor scope

SHOULD:
- Create separate monitors for stable vs variable services
- Set higher thresholds for services with known variability
- Change alert frequency from immediate to daily for noisy monitors

MAY:
- Implement custom filtering via SNS + Lambda to suppress known patterns
- Create anomaly suppression rules for planned events

## Common Issues

- symptoms: "Daily anomaly alerts for normal scaling"
  diagnosis: "Auto-scaling costs trigger anomaly detection."
  resolution: "Increase dollar threshold. Provide NO feedback on false positives."

- symptoms: "Monthly anomaly on RI/SP renewal"
  diagnosis: "Periodic RI/SP charges flagged as anomalies."
  resolution: "Provide feedback. ML model will learn recurring patterns over time."

- symptoms: "New service always flagged"
  diagnosis: "Insufficient baseline for new service spending."
  resolution: "Wait for baseline to establish. Provide feedback on expected costs."

## Output Format

```yaml
root_cause: "false_positives — <specific_cause>"
evidence:
  - type: anomaly_details
    content: "<false positive anomaly details>"
  - type: threshold_config
    content: "<current threshold and frequency settings>"
severity: LOW
mitigation:
  immediate: "Provide feedback and adjust thresholds"
  long_term: "Tune monitors and implement custom filtering"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — False positive management involves anomaly metadata |
| Blast Radius | LOW — Feedback and threshold changes affect alerting only |
| Rollback Complexity | LOW — Thresholds can be readjusted |
| Compliance Scope | LOW — Tuning anomaly detection is operational |

## Escalation Conditions

- Escalate to operations if false positive volume overwhelms the team and real anomalies are missed
- Escalate to finance if threshold increases risk missing genuine cost spikes

## Data Sensitivity

- **Anomaly feedback** is operational metadata
- **Threshold configurations** are operational settings

## PROHIBITED Actions

- Do NOT increase thresholds so high that genuine anomalies are suppressed
- Do NOT disable anomaly monitors to eliminate false positives — tune thresholds instead
- Do NOT ignore anomalies without investigation — verify they are truly false positives

## Phase 3 — Rollback

- Restore previous threshold settings if new thresholds miss genuine anomalies
- Reverse anomaly feedback if it was provided incorrectly
- Restore monitor configuration from documented settings

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
