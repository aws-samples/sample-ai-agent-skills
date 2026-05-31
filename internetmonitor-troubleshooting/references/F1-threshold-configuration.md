---
title: "F1 — Threshold Configuration"
description: "Diagnose health event threshold configuration issues"
status: active
severity: MEDIUM
triggers: ["too many health events", "no health events", "threshold wrong", "alert noise"]
owner: devops-agent
objective: "Resolve health event threshold configuration for appropriate alerting"
context: "Health event thresholds control when Internet Monitor generates health events. Default thresholds are availability < 96% and performance < 95%."
---

## Phase 1 — Triage

MUST:
- Check current threshold configuration: `aws internetmonitor get-monitor --monitor-name <name> --query 'HealthEventsConfig'`
- Review recent health events to assess alert volume: `aws internetmonitor list-health-events --monitor-name <name>`
- Determine if the issue is too many alerts (thresholds too sensitive) or too few (thresholds too lenient)

SHOULD:
- Compare current thresholds against baseline performance for the application
- Check if availability and performance thresholds are both configured
- Review the impact of current thresholds on operational noise

MAY:
- Analyze historical health events to determine optimal threshold values
- Compare thresholds with industry standards for the application type

## Phase 2 — Remediate

MUST:
- Adjust thresholds based on application requirements: `aws internetmonitor update-monitor --monitor-name <name> --health-events-config '{"AvailabilityScoreThreshold":<value>,"PerformanceScoreThreshold":<value>}'`
- Document the rationale for threshold changes
- Verify health events are generated appropriately after threshold changes

SHOULD:
- Set availability threshold based on application SLA requirements
- Set performance threshold based on user experience requirements
- Test threshold changes in a non-production monitor first

MAY:
- Configure different thresholds for different monitors based on application criticality
- Set up CloudWatch alarms on Internet Monitor metrics for additional alerting layers

## Common Issues

- symptoms: "Receiving too many health events for minor performance fluctuations"
  diagnosis: "Performance threshold may be set too high (too sensitive)."
  resolution: "Lower the performance threshold. Default is 95%. For applications tolerant of minor fluctuations, consider 90% or lower. Adjust: `aws internetmonitor update-monitor --monitor-name <name> --health-events-config '{\"PerformanceScoreThreshold\":90}'`"

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "list-health-events: GREEN — read-only event listing"
  - "update-monitor health-events-config: YELLOW — changes alerting behavior"
  - "cloudwatch put-metric-alarm: YELLOW — creates new alarm resource"
```

## Escalation Conditions

- Threshold changes do not affect health event generation as expected
- Application requires different thresholds for different city-networks (not supported)
- Health events generated but not matching actual user-reported issues
- Threshold configuration not persisting after update-monitor calls

## Data Sensitivity

- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- `aws internetmonitor list-health-events`: LOW — returns event metadata
- Threshold values: LOW — operational configuration data

## Prohibited Actions

- NEVER set thresholds to 0% to suppress all health events
- NEVER change thresholds during an active incident to hide alerts
- NEVER set thresholds without understanding the application's performance baseline
- NEVER remove health event configuration entirely from a production monitor

## Phase 3 — Rollback

1. Revert thresholds to previous values: `aws internetmonitor update-monitor --monitor-name <name> --health-events-config '{"AvailabilityScoreThreshold":<previous>,"PerformanceScoreThreshold":<previous>}'`
2. Verify health events resume normal generation patterns
3. If CloudWatch alarms were modified, restore original configurations
4. Document the threshold change history and rationale
5. Confirm operational teams are aware of threshold changes

## Output Format

```yaml
root_cause: "threshold_configuration — <specific_cause>"
evidence:
  - type: threshold_analysis
    content: "<current thresholds and health event patterns>"
severity: MEDIUM
mitigation:
  immediate: "Adjust thresholds to appropriate values"
  long_term: "Establish threshold review process aligned with SLA changes"
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
