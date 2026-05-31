---
title: "B1 — SLO Creation Issues"
description: "Diagnose CloudWatch Application Signals SLO creation failures"
status: active
severity: MEDIUM
triggers: ["SLO creation failed", "cannot create SLO", "service level objective error", "SLO configuration"]
owner: devops-agent
objective: "Resolve SLO creation and configuration issues"
context: "SLOs define service level targets based on Application Signals metrics (latency, availability)."
---

## Phase 1 — Triage

MUST:
- List existing SLOs: `aws application-signals list-service-level-objectives`
- Verify the target service exists in Application Signals: `aws application-signals list-services --start-time $(date -d '-1 hour' -u +%s) --end-time $(date -u +%s)`
- Check IAM permissions for application-signals:CreateServiceLevelObjective

SHOULD:
- Verify the SLI metric type is valid (latency or availability)
- Check that the service has sufficient metric data for SLO evaluation

MAY:
- Review existing SLOs for naming conflicts
- Check CloudTrail for CreateServiceLevelObjective errors

## Phase 2 — Remediate

MUST:
- Ensure the target service is discovered and emitting metrics
- Fix SLO configuration parameters (metric type, threshold, goal percentage, period)
- Grant application-signals:CreateServiceLevelObjective permission

SHOULD:
- Start with a conservative SLO target and tighten over time
- Verify metric data exists before creating the SLO

MAY:
- Create SLOs via CloudFormation for version control
- Set up multiple SLOs for different operations on the same service

## Common Issues

- symptoms: "CreateServiceLevelObjective returns validation error"
  diagnosis: "SLO configuration has invalid parameters (e.g., goal > 100%, invalid metric type)."
  resolution: "Verify goal is between 0 and 100. Use valid metric types: latency or availability. Check period is valid."

- symptoms: "SLO created but shows Insufficient Data immediately"
  diagnosis: "Target service is not emitting Application Signals metrics."
  resolution: "Verify service instrumentation. Check metrics in AWS/ApplicationSignals namespace. Allow 5-10 minutes for data."

## Safety Ratings

```
safety_ratings:
  - "list-service-level-objectives: GREEN — read-only SLO listing"
  - "list-services: GREEN — read-only service listing"
  - "create-service-level-objective: YELLOW — creates monitoring resource"
  - "delete-service-level-objective: YELLOW — removes monitoring resource"
```

## Escalation Conditions

- SLO creation fails with internal service errors after fixing configuration
- Service metrics exist but SLO evaluation consistently shows Insufficient Data
- SLO target requires custom metric types not supported by Application Signals
- Organization requires SLOs across multiple regions with consistent configuration

## Data Sensitivity

- `aws application-signals list-service-level-objectives`: LOW — returns SLO metadata
- `aws application-signals list-services`: LOW — returns service names
- SLO configuration: LOW — contains target percentages and metric types
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create SLOs with 100% availability targets — this is unrealistic and causes constant breach alerts
- NEVER delete SLOs to reset evaluation history without stakeholder approval
- NEVER create SLOs for services that are not yet instrumented
- NEVER set SLO evaluation periods shorter than the metric collection interval

## Phase 3 — Rollback

1. If an SLO was created with incorrect configuration, update it: `aws application-signals update-service-level-objective --slo-id <slo-id> --sli-config <corrected-config>`
2. If the SLO must be removed: `aws application-signals delete-service-level-objective --slo-id <slo-id>`
3. If alarms were created for the SLO, remove them as well
4. Verify remaining SLOs are unaffected by the change
5. Document the SLO configuration change and reason

## Output Format

```yaml
root_cause: "slo_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix SLO configuration"
  long_term: "Standardize SLO definitions via IaC"
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
