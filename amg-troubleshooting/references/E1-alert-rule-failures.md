---
title: "E1 — Alert Rule Failures"
description: "Diagnose alert rule evaluation failures in Amazon Managed Grafana"
status: active
severity: HIGH
triggers:
  - "alert not firing"
  - "alert rule error"
  - "alert evaluation failed"
  - "alert state pending"
  - "alerting not working"
owner: devops-agent
objective: "Identify why alert rules fail to evaluate or fire in AMG and restore alerting functionality"
context: "AMG uses Grafana unified alerting. Alert rules are defined independently from dashboards and evaluate queries on a schedule. Failures can stem from query errors, data source issues, incorrect conditions, evaluation group misconfiguration, or notification policy problems."
---

## Phase 1 — Triage

MUST:
- List alert rules via Grafana API: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/v1/provisioning/alert-rules`
- Check the alert rule state and last evaluation result
- Verify the data source used by the alert rule is functional
- Check the alert rule query and condition configuration

SHOULD:
- Test the alert rule query in the Explore view to verify it returns data
- Check the evaluation group interval and ensure it's appropriate
- Review notification policies: `curl -H "Authorization: Bearer <api-key>" https://<workspace-endpoint>/api/v1/provisioning/policies`

MAY:
- Check alert rule history for past evaluation results
- Verify contact points are configured and reachable

## Phase 2 — Remediate

MUST:
- Fix data source connectivity if the alert query fails
- Correct the alert condition (threshold, reduce expression, math expression)
- Ensure the evaluation group is active and the interval is appropriate

SHOULD:
- Test the alert rule with a condition that should trigger to verify end-to-end flow
- Configure appropriate "for" duration to avoid flapping alerts
- Set up a default notification policy to catch unrouted alerts

MAY:
- Use alert rule provisioning API for version-controlled alert management
- Set up silences for known maintenance windows

## Common Issues

- symptoms: "Alert rule shows 'Error' state"
  diagnosis: "The alert query fails to execute, typically due to data source permission or connectivity issues."
  resolution: "Fix the underlying data source issue. Check workspace IAM role permissions."

- symptoms: "Alert rule stays in 'Pending' state and never fires"
  diagnosis: "The 'for' duration is longer than the condition is true, or the evaluation interval is too long."
  resolution: "Reduce the 'for' duration or increase the evaluation frequency."

- symptoms: "Alert fires but no notification is received"
  diagnosis: "Notification policy does not route the alert to a contact point, or the contact point is misconfigured."
  resolution: "Check notification policies and contact point configuration. See E2 for notification channel issues."

## Safety Ratings

```
safety_ratings:
  - "list alert rules via API: GREEN — read-only enumeration"
  - "check notification policies via API: GREEN — read-only check"
  - "fix alert condition/threshold: YELLOW — changes when alerts fire"
  - "test alert with triggering condition: YELLOW — may send test notifications"
```

## Escalation Conditions

- Critical production alerts stopped firing, leaving systems unmonitored
- Alert rule errors affect multiple rules simultaneously
- Notification policy changes affect alert routing for the entire workspace
- Alert rules need reconfiguration after a Grafana version upgrade

## Data Sensitivity

- Alert rule configuration: MEDIUM — reveals monitoring thresholds and conditions
- Notification policies: MEDIUM — reveals alert routing and contact points
- Alert query results: HIGH — contain monitoring data from connected services

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER disable alert rules on production workspaces without a temporary alternative
- NEVER set alert evaluation intervals shorter than the data source scrape interval

## Phase 3 — Rollback

1. If alert rules were modified incorrectly, restore from provisioning API backup or manual reconfiguration
2. If notification policies were changed, restore the previous routing configuration via API
3. If alert conditions were changed, restore original thresholds and expressions
4. Verify alerts fire correctly by testing with a known-triggering condition
5. Confirm notification delivery end-to-end after rollback

## Output Format

```yaml
root_cause: "alert_rule_failure — <specific_cause>"
evidence:
  - type: alert_rule
    content: "<alert rule configuration and state>"
  - type: query_result
    content: "<alert query evaluation result>"
  - type: notification_policy
    content: "<notification routing configuration>"
severity: HIGH
mitigation:
  immediate: "Fix alert query, condition, or notification routing"
  long_term: "Implement alert rule testing and monitoring for alert health"
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
