---
title: "B2 — SLO Evaluation Errors"
description: "Diagnose CloudWatch Application Signals SLO evaluation and compliance issues"
status: active
severity: HIGH
triggers: ["SLO breach", "SLO insufficient data", "error budget exhausted", "SLO evaluation wrong"]
owner: devops-agent
objective: "Resolve SLO evaluation, compliance, and error budget issues"
context: "SLOs evaluate service performance over rolling windows. Evaluation depends on continuous metric flow."
---

## Phase 1 — Triage

MUST:
- Get SLO details: `aws application-signals get-service-level-objective --slo-id <slo-id>`
- Check SLO compliance status and error budget
- Verify metrics are flowing: `aws cloudwatch get-metric-statistics --namespace 'AWS/ApplicationSignals' --metric-name Latency --start-time $(date -d '-1 hour' -u +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 300 --statistics Average`

SHOULD:
- Check if the service experienced recent incidents affecting metrics
- Review the SLO evaluation window and remaining error budget

MAY:
- Compare SLO metrics with raw application metrics for discrepancies
- Check if metric collection gaps caused evaluation errors

## Phase 2 — Remediate

MUST:
- Fix metric collection issues if SLO shows Insufficient Data
- Investigate service degradation if SLO is breached
- Verify SLO threshold is appropriate for the service

SHOULD:
- Adjust SLO target if it was set too aggressively
- Set up alarms for error budget consumption rate

MAY:
- Create composite SLOs for multi-operation services
- Implement error budget policies for automated responses

## Common Issues

- symptoms: "SLO shows breach but service appears healthy"
  diagnosis: "SLO threshold is too aggressive or metric includes non-critical operations."
  resolution: "Review SLO target. Consider filtering to specific operations. Adjust threshold based on baseline performance."

- symptoms: "SLO shows Insufficient Data intermittently"
  diagnosis: "Metric collection has gaps due to agent restarts or scaling events."
  resolution: "Ensure CloudWatch agent and ADOT agent have stable deployments. Check for pod evictions or agent crashes."

## Safety Ratings

```
safety_ratings:
  - "get-service-level-objective: GREEN — read-only SLO details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric data"
  - "update-service-level-objective: YELLOW — modifies SLO configuration"
  - "Service remediation actions: RED — may affect production traffic"
```

## Escalation Conditions

- SLO breach indicates genuine service degradation requiring incident response
- Error budget is exhausted and further deployments should be frozen
- SLO evaluation shows inconsistent results despite stable metric flow
- Multiple SLOs breach simultaneously indicating systemic issues

## Data Sensitivity

- `aws application-signals get-service-level-objective`: LOW — returns SLO configuration
- `aws cloudwatch get-metric-statistics`: MEDIUM — reveals service performance data
- Error budget data: LOW — contains compliance percentages
- Service performance metrics: MEDIUM — reveals latency and error rate patterns

## Prohibited Actions

- NEVER adjust SLO targets to hide genuine service degradation
- NEVER delete and recreate SLOs to reset error budgets without approval
- NEVER ignore SLO breaches — they indicate service quality issues
- NEVER modify SLO evaluation windows during an active incident

## Phase 3 — Rollback

1. If SLO configuration was changed incorrectly, revert: `aws application-signals update-service-level-objective --slo-id <slo-id> --goal <original-goal>`
2. If service changes caused the SLO breach, roll back the service deployment
3. Verify metric flow resumes after any instrumentation changes
4. Confirm SLO evaluation returns to expected behavior
5. Document the SLO breach timeline and root cause

## Output Format

```yaml
root_cause: "slo_evaluation — <specific_cause>"
evidence:
  - type: slo_status
    content: "<evaluation details>"
severity: HIGH
mitigation:
  immediate: "Fix metric collection or service issue"
  long_term: "Implement error budget alerting"
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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
