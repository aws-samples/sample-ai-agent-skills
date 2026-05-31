---
title: "D3 — Predictive Scaling Issues"
description: "Diagnose predictive scaling forecast inaccuracies and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "predictive scaling not working"
  - "forecast inaccurate"
  - "predictive scaling not scaling"
  - "forecast not generated"
  - "predictive scaling wrong capacity"
owner: devops-agent
objective: "Identify and resolve predictive scaling issues"
context: "Predictive scaling uses ML to forecast demand and pre-provision capacity. Issues include: insufficient historical data, non-recurring traffic patterns, forecast mode set to ForecastOnly, scheduling buffer misconfigured, or metric specification wrong."
---

## Phase 1 — Triage

MUST:
- Check predictive scaling policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name> --policy-types PredictiveScaling`
- Check forecast: `aws autoscaling get-predictive-scaling-forecast --auto-scaling-group-name <asg-name> --policy-name <policy-name> --start-time <start> --end-time <end>`
- Verify policy mode (ForecastOnly vs ForecastAndScale): check Mode in policy configuration
- Check historical data availability in CloudWatch

SHOULD:
- Compare forecast vs actual capacity over past week
- Verify metric specification matches actual workload metric
- Check scheduling buffer time (SchedulingBufferTime)

MAY:
- Review if traffic patterns are sufficiently recurring
- Check if dynamic scaling is also configured as backup

## Phase 2 — Remediate

MUST:
- If insufficient data: wait for at least 24 hours (14 days recommended)
- If ForecastOnly mode: switch to ForecastAndScale after validating forecasts
- If metric wrong: update metric specification

SHOULD:
- Update to ForecastAndScale: `aws autoscaling put-scaling-policy --auto-scaling-group-name <asg-name> --policy-name predictive --policy-type PredictiveScaling --predictive-scaling-configuration '{"MetricSpecifications":[...],"Mode":"ForecastAndScale","SchedulingBufferTime":300}'`
- Combine with dynamic scaling for unexpected spikes

MAY:
- Use custom metric pairs for more accurate forecasting
- Adjust scheduling buffer for instance startup time

## Output Format

```yaml
root_cause: "predictive_scaling_issue — <specific_cause>"
evidence:
  - type: forecast_data
    content: "<forecast vs actual>"
  - type: policy_config
    content: "<predictive scaling configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix metric specification or switch mode"
  long_term: "Combine predictive with dynamic scaling for comprehensive coverage"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Wait for historical data accumulation | GREEN | No action — passive waiting |
| Switch to ForecastAndScale mode | YELLOW | Enables automated pre-provisioning |
| Update metric specification | YELLOW | Changes forecast accuracy basis |
| Combine with dynamic scaling | GREEN | Adds complementary scaling — non-destructive |
| Adjust scheduling buffer | YELLOW | Changes pre-provisioning timing |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- ForecastAndScale mode automatically changes capacity based on predictions
- Metric specification changes affect forecast accuracy

## Data Sensitivity

- Predictive scaling forecasts reveal expected traffic patterns
- Historical data reveals business activity cycles
- Metric specifications reveal key performance indicators

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER enable ForecastAndScale without validating forecasts in ForecastOnly mode first
- NEVER rely solely on predictive scaling without dynamic scaling backup

## Phase 3 — Rollback

- If ForecastAndScale was enabled prematurely: switch back to ForecastOnly mode
- If metric specification was changed: restore original metric configuration
- If scheduling buffer was set incorrectly: adjust to appropriate value
- Document all predictive scaling changes for operational audit trail

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
