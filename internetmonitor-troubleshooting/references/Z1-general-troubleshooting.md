---
title: "Z1 — General Internet Monitor Troubleshooting"
description: "Catch-all runbook for Internet Monitor issues"
status: active
severity: MEDIUM
triggers: ["Internet Monitor error", "internet monitor not working", "monitor issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Internet Monitor issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List monitors: `aws internetmonitor list-monitors`
- Get monitor details: `aws internetmonitor get-monitor --monitor-name <name>`
- Check for active health events: `aws internetmonitor list-health-events --monitor-name <name> --event-status ACTIVE`

SHOULD:
- Review CloudTrail for Internet Monitor API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=internetmonitor.amazonaws.com --max-results 20`
- Check monitor ProcessingStatus for data pipeline health
- Verify IAM permissions for the caller

MAY:
- Check AWS Health Dashboard for Internet Monitor service issues
- Review global internet events: `aws internetmonitor list-internet-events`

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook
- Document initial findings and investigation path

SHOULD:
- Verify monitor has resources attached and is in ACTIVE status
- Check if the issue is data-related, configuration-related, or integration-related

MAY:
- Set up monitoring for the monitor itself (meta-monitoring)
- Create CloudWatch alarms on Internet Monitor metrics

## Common Issues

- symptoms: "Internet Monitor not showing any data"
  diagnosis: "Monitor may have no resources, be in wrong status, or have insufficient traffic."
  resolution: "Check: 1) Monitor status is ACTIVE, 2) Resources are attached, 3) ProcessingStatus is OK, 4) Resources have active traffic."

## Safety Ratings

```
safety_ratings:
  - "list-monitors: GREEN — read-only enumeration"
  - "get-monitor: GREEN — read-only monitor details"
  - "list-health-events: GREEN — read-only event listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Internet Monitor service appears non-functional across multiple monitors
- No data generated despite confirmed active traffic through monitored resources
- CloudTrail shows unauthorized API calls to Internet Monitor
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws internetmonitor list-monitors`: LOW — returns monitor metadata
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- `aws internetmonitor list-health-events`: LOW — returns event metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete a monitor during general troubleshooting
- NEVER modify monitor configuration without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER make changes to monitored resources (CloudFront/VPC) to troubleshoot Internet Monitor

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore monitor settings if they were modified during troubleshooting
3. Verify all monitors are functioning normally before closing the investigation
4. Document all investigation steps and findings
5. Confirm monitored resources were not affected by the investigation

## Output Format

```yaml
root_cause: "general_internetmonitor — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting for Internet Monitor health"
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
