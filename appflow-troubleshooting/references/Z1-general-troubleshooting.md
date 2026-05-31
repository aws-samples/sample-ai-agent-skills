---
title: "Z1 — General AppFlow Troubleshooting"
description: "Catch-all runbook for Amazon AppFlow issues"
status: active
severity: MEDIUM
triggers: ["AppFlow error", "AppFlow not working", "AppFlow issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified AppFlow issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List flows: `aws appflow list-flows`
- Check for failed executions: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 10`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=appflow.amazonaws.com --max-results 20`

SHOULD:
- Check connector profile health: `aws appflow list-connector-profiles`
- Verify IAM permissions for the AppFlow service role

MAY:
- Check AWS Health Dashboard for AppFlow service issues
- Review CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/AppFlow --metric-name FlowExecutionsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up CloudWatch alarms for flow execution failures

## Safety Ratings

```
safety_ratings:
  - "list-flows: GREEN — read-only enumeration"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "list-connector-profiles: GREEN — read-only profile enumeration"
```

## Escalation Conditions

- AppFlow service appears non-functional across multiple flows
- No execution records despite active scheduled flows
- CloudTrail shows unauthorized API calls to AppFlow
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws appflow list-flows`: LOW — returns flow metadata
- `aws appflow describe-flow-execution-records`: MEDIUM — reveals execution details
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete flows or connector profiles during general troubleshooting
- NEVER modify production flow configurations without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER assume the issue is transient without checking execution history

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore flow settings if they were modified during troubleshooting
3. Verify all flows are in their expected state (ACTIVE/SUSPENDED)
4. Document all investigation steps and findings
5. Verify the AppFlow service is functioning normally before closing

## Common Issues

- symptoms: "Multiple flows failing simultaneously"
  diagnosis: "Shared connector profile has expired credentials or SaaS API is down."
  resolution: "Check connector profile health. Verify SaaS provider API status. Re-authenticate if needed."

## Output Format

```yaml
root_cause: "general_appflow — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
