---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Lookout for Metrics issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Lookout for Metrics error", "general issue", "unknown error", "service problem"]
owner: devops-agent
objective: "Systematically diagnose and resolve general Lookout for Metrics issues"
context: "This runbook covers issues that do not fit neatly into specific categories. It provides a systematic approach to diagnosing any Lookout for Metrics problem by checking common failure points in order of likelihood."
---

## Phase 1 — Triage

MUST:
- List all detectors and their status: `aws lookoutmetrics list-anomaly-detectors --query 'AnomalyDetectorSummaryList[*].{Name:AnomalyDetectorName,Arn:AnomalyDetectorArn,Status:Status}'`
- For affected detector, get full details: `aws lookoutmetrics describe-anomaly-detector --anomaly-detector-arn <arn>`
- Check data quality: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`

SHOULD:
- List metric sets: `aws lookoutmetrics list-metric-sets --anomaly-detector-arn <arn>`
- Check alerts: `aws lookoutmetrics list-alerts --anomaly-detector-arn <arn>`
- Review CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=lookoutmetrics.amazonaws.com --max-results 20`

MAY:
- Check service health dashboard for Lookout for Metrics outages
- Review IAM policies for recent changes affecting lookoutmetrics permissions
- Check service quotas: `aws service-quotas list-service-quotas --service-code lookoutmetrics`

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Follow the appropriate specific runbook (A1–F1) if the issue matches a known category
- Verify the fix resolves the issue

SHOULD:
- Document the root cause and resolution
- Check if the issue affects other detectors in the same account
- Review recent infrastructure changes that may have caused the issue

MAY:
- Implement monitoring to detect similar issues proactively
- Create CloudWatch alarms for Lookout for Metrics operational metrics
- Set up automated health checks for detector status

## Safety Ratings

```
safety_ratings:
  - "list-anomaly-detectors: GREEN — read-only enumeration"
  - "describe-anomaly-detector: GREEN — read-only status check"
  - "get-data-quality-metrics: GREEN — read-only quality check"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook after systematic triage
- Multiple detectors fail simultaneously suggesting a service-level issue
- Error messages reference internal service errors with no actionable information
- Issue persists after following all relevant specific runbooks

## Data Sensitivity

- `aws lookoutmetrics list-anomaly-detectors`: LOW — detector metadata only
- `aws lookoutmetrics describe-anomaly-detector`: MEDIUM — configuration details
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principals and actions
- Service quota information: LOW — account limits only

## Prohibited Actions

- NEVER delete detectors as a troubleshooting step without explicit approval
- NEVER modify production detector configurations without a rollback plan
- NEVER ignore data quality warnings — they are the most common root cause
- NEVER assume the issue is a service bug without exhausting configuration checks

## Phase 3 — Rollback

1. Revert any configuration changes made during troubleshooting
2. If a detector was deactivated for investigation, reactivate after confirming the fix
3. If IAM policies were modified, revert to previous versions
4. Document all troubleshooting steps and their outcomes
5. Notify stakeholders about any detection gaps during the investigation

## Common Issues

- symptoms: "Detector status is ACTIVE but no results visible"
  diagnosis: "Detector is still in the learning period (1–2 weeks after activation)."
  resolution: "Wait for the learning period to complete. Check activation date in detector details."

- symptoms: "API calls return ThrottlingException"
  diagnosis: "API rate limits exceeded due to too many concurrent requests."
  resolution: "Implement exponential backoff. Reduce API call frequency. Check for runaway automation."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details and diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified root cause"
  long_term: "Implement proactive monitoring and alerting"
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
