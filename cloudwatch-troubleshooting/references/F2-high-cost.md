---
title: "F2 — Contributor Insights High Cost"
description: "Diagnose unexpected Contributor Insights costs"
status: active
severity: MEDIUM
triggers:
  - "Contributor Insights cost"
  - "insight rule expensive"
  - "unexpected CloudWatch bill"
  - "matched log events cost"
owner: devops-agent
objective: "Identify and reduce unexpected Contributor Insights costs"
context: "Contributor Insights charges per matched log event (not per rule). A rule on a high-volume log group can generate significant costs. The first matched million events per month are included in the free tier. After that, charges apply per million matched events. Rules that match broadly on high-volume log groups are the primary cost driver."
---

## Phase 1 — Triage

MUST:
- List all active rules: `aws cloudwatch describe-insight-rules --query 'InsightRules[?State==ENABLED]'`
- Check which log groups each rule targets
- Estimate event volume for each targeted log group: `aws logs describe-log-groups --log-group-name-prefix <name> --query 'logGroups[].{name:logGroupName,storedBytes:storedBytes}'`
- Get rule reports to see matched event counts: `aws cloudwatch get-insight-rule-report --rule-name <name> --start-time <iso> --end-time <iso> --period 86400`

SHOULD:
- Check AWS Cost Explorer for Contributor Insights charges
- Identify rules with the highest matched event counts
- Check if rules have filters that could reduce matched events

MAY:
- Calculate cost: matched events beyond free tier × $0.02 per million (varies by region)
- Compare cost vs value of each rule

## Phase 2 — Remediate

MUST:
- Disable rules that are not providing value: `aws cloudwatch disable-insight-rules --rule-names <name>`
- Add filters to rules matching too broadly to reduce matched event count
- Remove rules targeting high-volume log groups if not needed

SHOULD:
- Set up billing alarms for CloudWatch costs
- Review rules monthly for cost optimization
- Use managed rules (built-in) where available instead of custom rules

MAY:
- Move analysis to CloudWatch Logs Insights for ad-hoc queries (no per-event charge)
- Consolidate multiple rules into fewer, more targeted rules

## Common Issues

- symptoms: "CloudWatch bill spike from Contributor Insights"
  diagnosis: "Rule matching all events on a high-volume log group (e.g., VPC Flow Logs, ALB access logs)."
  resolution: "Add filters to the rule or disable it. Use Logs Insights for ad-hoc analysis instead."

- symptoms: "Multiple rules on the same log group"
  diagnosis: "Each rule independently matches and charges for events."
  resolution: "Consolidate rules or disable redundant ones."

- symptoms: "Rule cost higher than expected despite filters"
  diagnosis: "Filters reduce contributors shown but still match and charge for all evaluated events."
  resolution: "Contributor Insights charges for matched events, not displayed contributors. Tighten the rule's scope."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Disable expensive rules | YELLOW | Stops monitoring; verify not needed first |
| Add filters to reduce matched events | GREEN | Cost optimization without losing rule |
| Remove rules on high-volume log groups | YELLOW | Stops monitoring for those log groups |

## Escalation Conditions

- Contributor Insights costs exceeding budget — escalate to cost management team
- Disabling rules that may be needed for security monitoring — escalate to security team
- Cost spike from new rules on high-volume log groups — escalate to rule creator

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudwatch describe-insight-rules` | MEDIUM | Exposes rule configuration |
| `aws cloudwatch get-insight-rule-report` | MEDIUM | Exposes matched event counts |
| Cost Explorer data | MEDIUM | Exposes account spending |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** disable Contributor Insights rules used for security monitoring to save costs

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Disabled rules | Re-enable: `aws cloudwatch enable-insight-rules --rule-names <name>` |
| Added filters to rules | Remove filters to restore original matching |
| Removed rules | Recreate the rules with original configuration |

## Output Format

```yaml
root_cause: "contributor_insights_cost — <specific_cause>"
evidence:
  - type: active_rules
    content: "<list of enabled rules and their log groups>"
  - type: matched_events
    content: "<matched event counts per rule>"
severity: MEDIUM
mitigation:
  immediate: "Disable expensive rules or add filters to reduce matched events"
  long_term: "Monthly cost review of Contributor Insights rules"
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
