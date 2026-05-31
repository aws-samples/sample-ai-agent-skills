---
title: "A2 — Flow Configuration"
description: "Diagnose Amazon AppFlow flow configuration issues"
status: active
severity: MEDIUM
triggers: ["flow misconfigured", "flow configuration error", "update flow failed"]
owner: devops-agent
objective: "Resolve flow configuration issues including triggers, mappings, and tasks"
context: "Flow configuration includes source/destination settings, field mappings, tasks (filters, transforms), and trigger configuration."
---

## Phase 1 — Triage

MUST:
- Describe the flow: `aws appflow describe-flow --flow-name <flow-name>`
- Check trigger configuration (on-demand, scheduled, event-driven)
- Verify field mappings match current source schema

SHOULD:
- Check flow execution history: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 5`
- Verify task configuration (filters, masks, validations)

MAY:
- Compare flow configuration with a known working flow
- Review CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/AppFlow --metric-name FlowExecutionsFailed --dimensions Name=FlowName,Value=<flow-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Update field mappings to match current source schema
- Fix trigger configuration (correct cron expression, event source)
- Ensure task chain is valid (filters before transforms)

SHOULD:
- Test with an on-demand execution before enabling schedule
- Document the correct configuration

MAY:
- Export flow configuration for version control

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow details"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "update-flow: YELLOW — modifies flow configuration"
  - "start-flow: YELLOW — triggers a flow execution"
```

## Escalation Conditions

- Flow configuration appears correct but executions consistently fail
- Source schema changes require complex mapping redesign
- Flow requires task types not supported by the current connector
- Configuration changes affect production data pipelines

## Data Sensitivity

- `aws appflow describe-flow`: MEDIUM — reveals source/destination configuration and credentials references
- `aws appflow describe-flow-execution-records`: MEDIUM — reveals execution details and data volumes
- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metrics only

## Prohibited Actions

- NEVER modify production flow configuration without testing in a non-production flow first
- NEVER remove field mappings without understanding downstream dependencies
- NEVER change trigger configuration during active business hours without approval
- NEVER disable flow tasks (filters, validations) as a troubleshooting shortcut

## Phase 3 — Rollback

1. If flow configuration was changed, revert to previous settings using update-flow
2. If trigger was modified, restore original schedule or event configuration
3. Verify downstream systems are not affected by configuration changes
4. Run a test execution to confirm the reverted configuration works
5. Document configuration changes and rollback steps

## Common Issues

- symptoms: "Flow runs but produces empty output"
  diagnosis: "Filter tasks are too restrictive or source query returns no records."
  resolution: "Review filter task conditions. Test with relaxed filters to confirm data availability."

## Output Format

```yaml
root_cause: "flow_configuration — <specific_cause>"
evidence:
  - type: configuration_detail
    content: "<config details>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration"
  long_term: "Version control flow configs"
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
