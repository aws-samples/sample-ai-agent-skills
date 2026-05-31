---
title: "A2 — Monitor Configuration"
description: "Diagnose Internet Monitor configuration issues"
status: active
severity: MEDIUM
triggers: ["monitor not collecting data", "monitor configuration wrong", "update monitor failed"]
owner: devops-agent
objective: "Resolve monitor configuration and update issues"
context: "Monitor configuration includes resources, MaxCityNetworksToMonitor, thresholds, and log delivery settings."
---

## Phase 1 — Triage

MUST:
- Get monitor details: `aws internetmonitor get-monitor --monitor-name <name>`
- Check monitor status and processing status fields
- Verify monitored resources list is correct and non-empty

SHOULD:
- Check MaxCityNetworksToMonitor value against expected traffic patterns
- Verify health event threshold configuration
- Check if log delivery is configured: look for S3Config and LogDeliveryConfig in monitor details

MAY:
- Compare current configuration against IaC templates (CloudFormation/Terraform)
- Review recent update-monitor calls in CloudTrail

## Phase 2 — Remediate

MUST:
- Update monitor with correct resources: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-add <arn>`
- Fix MaxCityNetworksToMonitor if set too low for traffic patterns
- Ensure monitor status is ACTIVE

SHOULD:
- Configure appropriate health event thresholds for availability and performance
- Set up log delivery if monitoring data export is needed

MAY:
- Adjust traffic percentage monitored for cost optimization
- Update tags for cost allocation tracking

## Common Issues

- symptoms: "Monitor shows INACTIVE status with no health events"
  diagnosis: "No resources attached to the monitor or resources have no traffic."
  resolution: "Add resources: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-add <arn>`. Verify resources have active traffic."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details retrieval"
  - "update-monitor: YELLOW — modifies monitor configuration, may affect cost"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "describe-vpcs: GREEN — read-only VPC enumeration"
```

## Escalation Conditions

- Monitor stuck in non-ACTIVE status despite correct configuration
- ProcessingStatus shows persistent errors not resolved by configuration changes
- Configuration changes not taking effect after reasonable processing time
- Monitor configuration drift detected between IaC and actual state

## Data Sensitivity

- `aws internetmonitor get-monitor`: LOW — returns monitor configuration metadata
- `aws internetmonitor update-monitor`: MEDIUM — modifies monitoring scope and thresholds
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER remove all resources from a production monitor without explicit approval
- NEVER reduce MaxCityNetworksToMonitor on a production monitor without analyzing traffic impact
- NEVER disable log delivery on a monitor that feeds downstream alerting systems
- NEVER change health event thresholds without documenting the previous values

## Phase 3 — Rollback

1. Revert resource changes: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-add <removed-arn> --resources-to-remove <added-arn>`
2. Restore MaxCityNetworksToMonitor to previous value if changed
3. Restore threshold configuration to previous values
4. Verify monitor returns to ACTIVE status with correct ProcessingStatus
5. Document all configuration changes and rollback actions

## Output Format

```yaml
root_cause: "monitor_configuration — <specific_cause>"
evidence:
  - type: configuration_issue
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix monitor configuration"
  long_term: "Manage monitor configuration via IaC"
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
