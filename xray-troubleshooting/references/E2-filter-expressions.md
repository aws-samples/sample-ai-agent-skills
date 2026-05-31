---
title: "E2 — Filter Expression Issues"
description: "Diagnose X-Ray filter expression syntax and matching problems"
status: active
severity: LOW
triggers:
  - "filter expression error"
  - "filter not matching"
  - "X-Ray query syntax"
  - "filter expression syntax"
owner: devops-agent
objective: "Identify and fix filter expression syntax and matching issues"
context: "X-Ray filter expressions use a specific syntax for querying traces. They support service names, URLs, response times, status codes, annotations, and boolean operators. Incorrect syntax causes errors or empty results."
---

## Phase 1 — Triage

MUST:
- Test the filter expression: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression '<expression>'`
- Verify the syntax follows X-Ray filter expression language
- Check if the filter criteria match actual trace data
- Test with a simple expression first, then add complexity

SHOULD:
- Verify annotation keys and values used in the expression
- Check if service names are correctly quoted
- Verify numeric comparisons use correct operators

MAY:
- Review X-Ray filter expression documentation for syntax reference
- Test expressions in the X-Ray console for interactive feedback

## Phase 2 — Remediate

MUST:
- Fix filter expression syntax errors
- Use correct quoting for string values: `service("my-service")`
- Use correct operators for numeric values: `responsetime > 5`

SHOULD:
- Use annotations for custom filtering: `annotation.key = "value"`
- Combine expressions with AND/OR: `service("api") AND responsetime > 2`

MAY:
- Save commonly used expressions as X-Ray groups
- Document filter expression patterns for the team

## Common Issues

- symptoms: "Filter expression syntax error"
  diagnosis: "Incorrect quoting, operators, or field names."
  resolution: "Use double quotes for strings. Valid fields: service, edge, annotation, responsetime, etc."

- symptoms: "Filter matches nothing despite traces existing"
  diagnosis: "Service name or annotation key doesn't match actual trace data."
  resolution: "Check actual service names and annotation keys in existing traces."

- symptoms: "Complex filter expression not working"
  diagnosis: "Boolean operator precedence or nesting issue."
  resolution: "Use parentheses for complex expressions: (service(\"a\") OR service(\"b\")) AND responsetime > 1."

## Output Format

```yaml
root_cause: "filter_expression_issue — <specific_cause>"
evidence:
  - type: expression
    content: "<filter expression>"
  - type: syntax_error
    content: "<error message if any>"
  - type: matching_traces
    content: "<traces matching corrected expression>"
severity: LOW
mitigation:
  immediate: "Fix filter expression syntax"
  long_term: "Document and standardize filter expression patterns"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
