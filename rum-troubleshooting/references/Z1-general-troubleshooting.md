---
title: "Z1 — General RUM Troubleshooting"
description: "Catch-all runbook for CloudWatch RUM issues"
status: active
severity: MEDIUM
triggers: ["RUM error", "RUM not working", "real user monitoring issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified RUM issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List app monitors: `aws rum list-app-monitors`
- Get app monitor details: `aws rum get-app-monitor --name <name>`
- Check for recent data: `aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 5`

SHOULD:
- Review CloudTrail for RUM API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rum.amazonaws.com --max-results 20`
- Verify Cognito Identity pool configuration
- Check IAM permissions for the caller

MAY:
- Check AWS Health Dashboard for RUM service issues
- Verify the JavaScript snippet is installed in the application

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook
- Document initial findings and investigation path

SHOULD:
- Verify app monitor is ACTIVE
- Check if the issue is data collection, configuration, integration, or analysis related

MAY:
- Set up monitoring for RUM health
- Create CloudWatch alarms on RUM metrics

## Common Issues

- symptoms: "RUM not showing any data"
  diagnosis: "Multiple factors can prevent data collection: snippet missing, Cognito misconfigured, CSP blocking, sample rate 0."
  resolution: "Check in order: 1) App monitor ACTIVE, 2) JavaScript snippet installed, 3) Cognito Identity pool configured, 4) CSP allows RUM endpoints, 5) Session sample rate > 0."

## Safety Ratings

```
safety_ratings:
  - "list-app-monitors: GREEN — read-only enumeration"
  - "get-app-monitor: GREEN — read-only monitor details"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- RUM service appears non-functional across multiple app monitors
- No data collected despite confirmed snippet installation and Cognito configuration
- CloudTrail shows unauthorized API calls to RUM
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws rum list-app-monitors`: LOW — returns app monitor metadata
- `aws rum get-app-monitor`: MEDIUM — reveals application domain and monitoring configuration
- `aws rum get-app-monitor-data`: MEDIUM — contains user session data
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete an app monitor during general troubleshooting
- NEVER modify Cognito Identity pool settings without checking impact on other applications
- NEVER dismiss the issue without documenting the investigation path
- NEVER make configuration changes during general troubleshooting without identifying root cause first

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore app monitor, Cognito, or IAM settings if modified
3. Verify data collection is functioning normally before closing
4. Document all investigation steps and findings
5. Confirm the JavaScript snippet and Cognito integration are unaffected

## Output Format

```yaml
root_cause: "general_rum — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting for RUM health"
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
