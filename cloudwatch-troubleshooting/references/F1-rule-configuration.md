---
title: "F1 — Contributor Insights Rule Configuration"
description: "Diagnose Contributor Insights rule matching and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "Contributor Insights"
  - "insight rule"
  - "rule not matching"
  - "top contributors"
  - "insight rule configuration"
owner: devops-agent
objective: "Identify why a Contributor Insights rule is not matching expected log events"
context: "Contributor Insights rules analyze log data to find top contributors (e.g., top IP addresses, top error codes). Rules define a log group, log format (JSON or CLF), key fields, value fields, and optional filters. Rules must match the actual log format and field names."
---

## Phase 1 — Triage

MUST:
- Check rule configuration: `aws cloudwatch describe-insight-rules --query 'InsightRules[?Name==<name>]'`
- Verify the rule's log group has data in the evaluation period
- Check that the log format (JSON/CLF) matches the actual log format
- Verify field names in the rule match actual log event fields

SHOULD:
- Check rule state (ENABLED vs DISABLED): `aws cloudwatch describe-insight-rules`
- Test with a sample log event to verify field extraction
- Check if filters in the rule are too restrictive

MAY:
- Get rule report to see if any data is being matched: `aws cloudwatch get-insight-rule-report --rule-name <name> --start-time <iso> --end-time <iso> --period 3600`
- Compare with a known-working rule on the same log group

## Phase 2 — Remediate

MUST:
- Fix log format specification (JSON vs CLF) to match actual logs
- Fix field names to match actual log event structure
- Enable the rule if disabled

SHOULD:
- Relax filters to verify the rule matches any events, then tighten
- Use JSON log format when possible (more reliable field extraction)
- Test the rule with get-insight-rule-report after changes

MAY:
- Create a managed rule from the console for common patterns
- Document rule configurations for the operations team

## Common Issues

- symptoms: "Rule shows no contributors"
  diagnosis: "Log format mismatch or field names don't match actual log structure."
  resolution: "Verify log format (JSON/CLF). Check field names against actual log events."

- symptoms: "Rule matches fewer events than expected"
  diagnosis: "Filter in the rule is too restrictive."
  resolution: "Remove or relax the filter. Verify filter field values match actual data."

- symptoms: "Rule is DISABLED"
  diagnosis: "Rule was disabled manually or by automation."
  resolution: "Enable the rule: aws cloudwatch enable-insight-rules --rule-names <name>"

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix log format specification | GREEN | Rule configuration fix |
| Fix field names | GREEN | Rule configuration fix |
| Enable disabled rule | GREEN | Activates event matching |

## Escalation Conditions

- Contributor Insights rules for security monitoring not matching — escalate to security operations
- Rule configuration changes affecting production dashboards — escalate to dashboard owner
- Log format changes breaking existing rules — escalate to application team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-insight-rules` | MEDIUM | Exposes rule configuration |
| `aws cloudwatch get-insight-rule-report` | MEDIUM | Exposes top contributor data |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** disable Contributor Insights rules used for security monitoring without approval

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed rule configuration | Restore previous rule configuration |
| Enabled rule | Disable: `aws cloudwatch disable-insight-rules --rule-names <name>` |

## Output Format

```yaml
root_cause: "contributor_insights_rule — <specific_cause>"
evidence:
  - type: rule_config
    content: "<rule definition>"
  - type: rule_report
    content: "<get-insight-rule-report results>"
severity: MEDIUM
mitigation:
  immediate: "Fix log format, field names, or filters"
  long_term: "Validate rules against sample log events before deployment"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
