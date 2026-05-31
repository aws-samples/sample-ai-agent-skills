---
title: "D1 — Rule Evaluation Failures"
description: "Diagnose and resolve AMP ruler rule evaluation failures"
status: active
severity: HIGH
triggers:
  - "rule evaluation"
  - "recording rule"
  - "alerting rule"
  - "ruler"
  - "rule group"
  - "rules not working"
owner: devops-agent
objective: "Identify and fix rule evaluation failures in AMP ruler"
context: "AMP ruler evaluates recording rules and alerting rules on a configured interval. Rules are uploaded via aws amp put-rule-groups-namespace. Failures can be caused by invalid rule syntax, missing metrics, PromQL errors in rule expressions, or configuration upload failures."
---

## Phase 1 — Triage

MUST:
- List rule group namespaces: `aws amp list-rule-groups-namespaces --workspace-id <workspace-id>`
- Describe specific rule group: `aws amp describe-rule-groups-namespace --workspace-id <workspace-id> --name <namespace>`
- Check rule group status in the describe output — look for `status.statusCode`
- Validate rule file syntax locally: `promtool check rules <rules-file.yaml>`

SHOULD:
- Check CloudTrail for PutRuleGroupsNamespace errors
- Verify the metrics referenced in rules actually exist in the workspace
- Check if rule evaluation interval is appropriate for the data
- Test rule expressions as ad-hoc queries to verify they return results

MAY:
- Check CloudWatch for ruler-related errors
- Compare rule configuration against Prometheus rule format documentation
- Review rule group size — very large groups may have evaluation issues

## Phase 2 — Remediate

MUST:
- If rule syntax invalid: fix YAML syntax and re-upload:
  ```
  aws amp put-rule-groups-namespace --workspace-id <workspace-id> \
    --name <namespace> --data file://rules.yaml
  ```
- If PromQL expression invalid: fix the expression and validate with promtool
- If metrics missing: verify collectors are sending the referenced metrics
- If upload failed: check IAM permissions for `aps:PutRuleGroupsNamespace`

SHOULD:
- Use promtool to validate rules before uploading
- Keep rule groups focused — separate recording rules from alerting rules
- Set appropriate evaluation intervals (not too frequent for expensive queries)

MAY:
- Implement CI/CD pipeline for rule validation and deployment
- Version control all rule configurations
- Set up monitoring for rule evaluation health

## Common Issues

- symptoms: "PutRuleGroupsNamespace returns ValidationException"
  diagnosis: "Rule file has invalid YAML syntax or invalid PromQL expressions."
  resolution: "Validate with promtool check rules. Fix syntax errors and re-upload."

- symptoms: "Rule group status shows CREATION_FAILED or UPDATE_FAILED"
  diagnosis: "Rule configuration was rejected by the ruler."
  resolution: "Check the status reason in describe output. Fix the rule and re-upload."

- symptoms: "Recording rules not producing expected metrics"
  diagnosis: "Rule expression references metrics that don't exist or returns empty results."
  resolution: "Test the rule expression as an ad-hoc query. Verify source metrics are being ingested."

- symptoms: "Alerting rules not firing despite conditions being met"
  diagnosis: "Rule expression may not match expected data, or for/pending duration not elapsed."
  resolution: "Test the alert expression as a query. Check the 'for' duration. Verify alert manager is configured."

## Safety Ratings

```
safety_ratings:
  - "list-rule-groups-namespaces: GREEN — read-only enumeration"
  - "describe-rule-groups-namespace: GREEN — read-only config check"
  - "promtool check rules: GREEN — local validation"
  - "put-rule-groups-namespace: YELLOW — uploads new rule configuration"
```

## Escalation Conditions

- Alerting rules not firing, leaving production systems unmonitored
- Recording rules producing incorrect data used by dashboards
- Rule upload failures block CI/CD pipeline for rule deployment
- Rule evaluation consuming excessive resources in the workspace

## Data Sensitivity

- Rule group configuration: MEDIUM — reveals alerting thresholds and recording logic
- Rule evaluation results: MEDIUM — contain monitoring data
- SNS topic ARNs in alert rules: MEDIUM — reveal notification infrastructure

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER delete rule group namespaces without backing up the configuration first
- NEVER set evaluation intervals shorter than the data scrape interval

## Phase 3 — Rollback

1. If rule configuration was uploaded incorrectly, restore from version control: `aws amp put-rule-groups-namespace --workspace-id <id> --name <namespace> --data file://previous-rules.yaml`
2. If a rule group namespace was deleted, recreate it from backup
3. If rule expressions were changed, restore original expressions and re-upload
4. Verify alert rules are evaluating correctly after rollback
5. Confirm recording rules produce expected metrics

## Output Format

```yaml
root_cause: "Rule evaluation failure — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule group namespace status and configuration>"
  - type: promtool_validation
    content: "<validation results>"
severity: HIGH
mitigation:
  immediate: "Fix rule syntax or expression and re-upload"
  long_term: "Implement CI/CD validation pipeline for rules, version control configurations"
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
