---
title: "C1 — Flow Execution Failures"
description: "Diagnose Amazon AppFlow flow execution failures"
status: active
severity: HIGH
triggers: ["flow execution failed", "flow run error", "execution error"]
owner: devops-agent
objective: "Resolve flow execution failures including source errors, destination errors, and data processing issues"
context: "Flow executions can fail at source read, data processing, or destination write stages. Each stage has different error patterns."
---

## Phase 1 — Triage

MUST:
- Check execution records: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 10`
- Identify failure stage (source, processing, destination) from execution status
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/AppFlow --metric-name FlowExecutionsFailed --dimensions Name=FlowName,Value=<flow-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- Review execution error messages for specific error codes
- Check connector profile health: `aws appflow describe-connector-profiles --connector-profile-names <profile>`

MAY:
- Review CloudWatch Logs if logging is enabled: `aws logs filter-log-events --log-group-name /aws/appflow/<flow-name> --start-time <epoch-ms> --limit 50`

## Phase 2 — Remediate

MUST:
- Fix the identified failure stage (source auth, data mapping, destination permissions)
- Verify connector profile credentials are valid
- Ensure destination has sufficient capacity and permissions

SHOULD:
- Run a test execution with a small data subset
- Enable CloudWatch Logs for detailed execution logging

MAY:
- Implement error handling with flow notifications via SNS

## Safety Ratings

```
safety_ratings:
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "logs filter-log-events: GREEN — read-only log review"
  - "start-flow: YELLOW — triggers a new flow execution"
```

## Escalation Conditions

- Executions fail consistently with internal service errors
- Source API returns errors not documented in AppFlow error codes
- Destination write failures cause data inconsistency
- Execution failures affect SLA-bound data pipelines

## Data Sensitivity

- `aws appflow describe-flow-execution-records`: MEDIUM — reveals execution details and error messages
- `aws logs filter-log-events`: HIGH — may contain data samples and detailed error context
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only

## Prohibited Actions

- NEVER retry failed executions without understanding the root cause
- NEVER disable data validation tasks to bypass execution errors
- NEVER modify destination schema to match incorrect source data
- NEVER ignore partial execution failures — they may cause data gaps

## Phase 3 — Rollback

1. If partial data was written to destination, identify and clean up incomplete records
2. If flow configuration was changed during troubleshooting, revert to previous settings
3. For S3 destinations, remove partial output files from the execution prefix
4. For Redshift destinations, delete partially loaded records using execution timestamp
5. Document the failure, root cause, and remediation for post-incident review

## Common Issues

- symptoms: "Execution status shows Error with 'ThrottlingException' in error message"
  diagnosis: "Source API rate limit exceeded. Common with Salesforce during peak hours."
  resolution: "Schedule flows during off-peak hours. Reduce batch size. Use incremental transfers."

## Output Format

```yaml
root_cause: "flow_execution — <specific_cause>"
evidence:
  - type: execution_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix execution failure"
  long_term: "Implement monitoring and alerting"
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
