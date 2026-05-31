---
title: "F1 — Anomaly Detection Issues"
description: "Diagnose X-Ray Insights anomaly detection problems"
status: active
severity: MEDIUM
triggers:
  - "insights not working"
  - "anomaly not detected"
  - "X-Ray insights"
  - "anomaly detection"
owner: devops-agent
objective: "Identify and resolve X-Ray Insights anomaly detection issues"
context: "X-Ray Insights uses machine learning to detect anomalies in trace data. Insights must be enabled on X-Ray groups. Sufficient trace volume is needed for baseline establishment. Insights detect latency and error rate anomalies."
---

## Phase 1 — Triage

MUST:
- Check if insights are enabled: `aws xray get-groups` (look for InsightsConfiguration)
- List active insights: `aws xray get-insight-summaries --start-time <start> --end-time <end> --states ACTIVE`
- Verify sufficient trace volume exists for the group
- Check if the group's filter expression matches the expected services

SHOULD:
- Verify the group has been active long enough for baseline establishment
- Check if notifications are configured for insights
- Review recent insight history

MAY:
- Get insight details: `aws xray get-insight --insight-id <insight-id>`
- Check CloudWatch for X-Ray insight metrics

## Phase 2 — Remediate

MUST:
- Enable insights on the group: `aws xray update-group --group-name <name> --insights-configuration InsightsEnabled=true,NotificationsEnabled=true`
- Ensure sufficient trace volume for baseline establishment
- Configure notifications for insight alerts

SHOULD:
- Create groups for services that need anomaly detection
- Allow time for baseline establishment (typically 24-48 hours)

MAY:
- Set up SNS notifications for insights
- Create CloudWatch dashboards for insight monitoring

## Common Issues

- symptoms: "Insights enabled but no anomalies detected"
  diagnosis: "Insufficient trace volume or baseline not yet established."
  resolution: "Ensure adequate sampling rates. Allow 24-48 hours for baseline."

- symptoms: "Cannot enable insights on group"
  diagnosis: "Group quota or configuration issue."
  resolution: "Check group configuration. Insights require a valid filter expression."

- symptoms: "Insights generating too many false positives"
  diagnosis: "Normal traffic patterns being flagged as anomalies."
  resolution: "Review and adjust the group filter expression to focus on relevant services."

## Output Format

```yaml
root_cause: "anomaly_detection_issue — <specific_cause>"
evidence:
  - type: insights_config
    content: "<insights configuration on groups>"
  - type: active_insights
    content: "<current active insights>"
  - type: trace_volume
    content: "<trace volume for the group>"
severity: MEDIUM
mitigation:
  immediate: "Enable insights and ensure adequate trace volume"
  long_term: "Configure insights for all critical service groups"
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

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
