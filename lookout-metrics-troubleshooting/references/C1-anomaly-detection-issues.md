---
title: "C1 — Anomaly Detection Issues"
description: "Diagnose and resolve anomaly detection failures and missed anomalies"
status: active
severity: HIGH
triggers: ["no anomalies detected", "anomaly detection not working", "detector not finding anomalies", "missed anomaly"]
owner: devops-agent
objective: "Ensure the detector is correctly identifying anomalies in business metrics"
context: "Anomaly detection depends on data quality, learning period completion, sensitivity threshold, and proper metric set configuration. Most detection issues trace back to data quality rather than model problems."
---

## Phase 1 — Triage

MUST:
- Check detector status and learning period: `aws lookoutmetrics describe-anomaly-detector --anomaly-detector-arn <arn>`
- Check data quality metrics: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`
- List recent anomaly groups: `aws lookoutmetrics list-anomaly-group-summaries --anomaly-detector-arn <arn> --sensitivity-threshold 50`

SHOULD:
- Verify the detector has completed the learning period (1–2 weeks after activation)
- Check for data gaps or missing data points in the source
- Review sensitivity threshold setting

MAY:
- Compare anomaly results at different sensitivity thresholds
- Check if the metric values have sufficient variance for anomaly detection

## Phase 2 — Remediate

MUST:
- Wait for learning period completion if detector was recently activated
- Fix data quality issues identified by `get-data-quality-metrics`
- Adjust sensitivity threshold if too low (missing anomalies) or too high (too many false positives)

SHOULD:
- Ensure continuous data delivery without gaps
- Verify measure values have meaningful variance (constant values produce no anomalies)
- Check dimension cardinality is within expected range

MAY:
- Run back-test mode to validate detection on historical data
- Create a separate test detector with known anomalous data to verify service functionality

## Safety Ratings

```
safety_ratings:
  - "describe-anomaly-detector: GREEN — read-only status check"
  - "get-data-quality-metrics: GREEN — read-only quality check"
  - "list-anomaly-group-summaries: GREEN — read-only anomaly listing"
  - "Sensitivity threshold adjustment: YELLOW — changes detection behavior"
```

## Escalation Conditions

- Detector has completed learning period but produces zero anomalies on known anomalous data
- Data quality metrics show persistent issues despite data pipeline fixes
- Detector status transitions to FAILED during normal operation
- Anomaly detection accuracy degrades significantly over time

## Data Sensitivity

- `aws lookoutmetrics get-data-quality-metrics`: LOW — statistical summaries only
- `aws lookoutmetrics list-anomaly-group-summaries`: MEDIUM — reveals business metric anomaly patterns
- `aws lookoutmetrics get-anomaly-group`: MEDIUM — detailed anomaly information with metric values
- Sensitivity threshold configuration: LOW — operational parameter only

## Prohibited Actions

- NEVER conclude detection is broken during the learning period (first 1–2 weeks)
- NEVER set sensitivity to 100 in production without understanding false positive implications
- NEVER deactivate a production detector to "reset" detection — this destroys the learned model
- NEVER dismiss data quality issues as unrelated to detection problems

## Phase 3 — Rollback

1. If sensitivity was changed and results worsened, revert to previous threshold value
2. If detector was deactivated for troubleshooting, reactivate and plan for learning period reset
3. If data pipeline changes caused new issues, revert pipeline configuration
4. Document all changes to detection configuration
5. Notify stakeholders about any detection gaps during troubleshooting

## Common Issues

- symptoms: "No anomalies detected after 3+ weeks of operation"
  diagnosis: "Data quality issues — missing data points, constant values, or insufficient variance."
  resolution: "Check `get-data-quality-metrics`. Fix data pipeline to ensure continuous, variable data delivery."

- symptoms: "Detector was working but stopped detecting anomalies"
  diagnosis: "Data source connectivity issue or data pipeline failure causing missing data."
  resolution: "Verify data source connectivity. Check data pipeline health. Review recent infrastructure changes."

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: detection_issue
    content: "<data quality metrics and detector status>"
severity: HIGH
mitigation:
  immediate: "Fix data quality or sensitivity configuration"
  long_term: "Implement data quality monitoring and alerting"
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
