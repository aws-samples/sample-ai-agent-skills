---
title: "B3 — Anomaly Detection Issues"
description: "Diagnose Detective anomaly detection problems including false positives and missing anomalies"
status: active
severity: MEDIUM
triggers:
  - "anomaly detection"
  - "false positive"
  - "missing anomaly"
  - "behavioral baseline"
  - "detective anomaly"
owner: devops-agent
objective: "Resolve anomaly detection issues and improve detection accuracy"
context: "Detective uses machine learning to establish behavioral baselines and detect anomalies. It requires sufficient historical data (2+ weeks) to build accurate baselines. Common issues include insufficient data history, noisy baselines, and false positives."
---

## Phase 1 — Triage

MUST:
- Verify behavior graph has been active for at least 2 weeks (baseline building)
- Check data source coverage for the affected account
- Verify the entity has consistent activity history
- Review investigation indicators for anomaly context

SHOULD:
- Check if recent infrastructure changes explain the anomaly
- Verify data sources have been continuously enabled (gaps affect baselines)
- Review the time period of the anomaly

MAY:
- Compare anomaly with known deployment or change events
- Check if multiple entities show similar anomalies (may indicate legitimate change)

## Phase 2 — Remediate

MUST:
- If graph is new (< 2 weeks), wait for baseline to build
- If data sources were recently enabled, wait for sufficient history
- Document false positives for future reference

SHOULD:
- Correlate anomalies with change management records
- Use investigation features to understand anomaly context

## Output Format

```yaml
root_cause: "anomaly_detection — <specific_cause>"
evidence:
  - type: graph_age
    content: "<behavior graph creation date>"
  - type: anomaly_details
    content: "<anomaly indicators>"
severity: MEDIUM
mitigation:
  immediate: "Verify anomaly context and data source coverage"
  long_term: "Maintain continuous data source coverage for accurate baselines"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Anomaly data reveals unusual security-relevant behavior |
| Blast Radius | LOW — Anomaly detection is read-only analysis |
| Rollback Complexity | LOW — Baselines rebuild automatically over time |
| Compliance Scope | MEDIUM — Anomaly detection supports threat detection requirements |

## Escalation Conditions

- Escalate to incident response if anomalies indicate potential security compromise
- Escalate to security team if anomaly detection is not functioning due to data gaps

## Data Sensitivity

- **Anomaly indicators** reveal unusual behavior patterns — security-sensitive
- **Behavioral baselines** contain historical activity patterns

## PROHIBITED Actions

- Do NOT dismiss anomalies without investigation — they may indicate real threats
- Do NOT share anomaly details outside the security team
- Do NOT disable data sources that feed anomaly detection without security approval

## Phase 3 — Rollback

- No rollback needed — anomaly detection is automatic
- If baselines are inaccurate due to data gaps, wait for continuous data to rebuild baselines (2+ weeks)
- Document any periods of reduced anomaly detection capability

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
