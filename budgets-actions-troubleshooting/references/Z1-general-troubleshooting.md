---
title: "Z1 — General Troubleshooting"
description: "General AWS Budgets Actions troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["budget error", "budget issue", "budget problem", "budgets help"]
owner: devops-agent
objective: "Diagnose and resolve general Budgets Actions issues"
context: "Catch-all runbook for Budgets Actions issues that do not fit specific categories. Routes to appropriate specific runbooks when possible."
---

## Phase 1 — Triage

MUST:
- Identify the Budgets component involved (budget, action, notification, threshold, organization)
- Check CloudTrail for recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=budgets.amazonaws.com --max-results 20`
- Verify IAM permissions: `aws sts get-caller-identity`

SHOULD:
- List all budgets and actions: `aws budgets describe-budgets --account-id <account-id>`
- Check AWS Health Dashboard for Budgets service issues: `aws health describe-events --filter '{"services":["BUDGETS"]}'`

MAY:
- Check AWS Status Page for regional issues
- Review Budgets documentation for recent changes

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- Fix IAM permissions if they are the root cause
- Document the issue for future reference

SHOULD:
- Open AWS Support case if the issue is service-side
- Check for known issues in AWS documentation

MAY:
- Implement monitoring for the identified issue type
- Set up CloudWatch alarms for Budgets-related errors

## Safety Ratings

```
safety_ratings:
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "budgets describe-budgets: GREEN — read-only listing"
  - "health describe-events: GREEN — read-only service health check"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook
- Multiple Budgets components are affected simultaneously
- Issue appears to be a service-side problem
- Issue affects cost control for production workloads

## Data Sensitivity

- `aws cloudtrail lookup-events`: MEDIUM — may reveal user actions
- `aws sts get-caller-identity`: LOW — returns caller identity
- Budget data: MEDIUM — reveals spending limits and patterns
- Health events: LOW — returns service status

## Prohibited Actions

- NEVER make changes without identifying the root cause first
- NEVER assume the issue is on the AWS side without evidence
- NEVER share budget details in public forums
- NEVER ignore recurring issues — document and track patterns

## Phase 3 — Rollback

1. Revert any changes made during troubleshooting
2. Restore original IAM permissions if modified
3. Cancel any pending operations started during investigation
4. Verify the system is in the same state as before troubleshooting
5. Document findings and escalation path

## Common Issues

- symptoms: "General Budgets errors with unclear cause"
  diagnosis: "Multiple potential causes — requires systematic triage."
  resolution: "Follow Phase 1 triage to identify the component, then route to the specific runbook."

## Output Format

```yaml
root_cause: "general_budgets — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
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
