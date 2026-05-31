---
title: "D2 — Trace Analysis Issues"
description: "Diagnose problems with X-Ray trace analysis and querying"
status: active
severity: MEDIUM
triggers:
  - "trace analysis not working"
  - "cannot find traces"
  - "filter expression error"
  - "trace query issues"
owner: devops-agent
objective: "Identify and resolve trace analysis and querying issues"
context: "X-Ray trace analysis uses filter expressions to find specific traces. Traces can be filtered by service, URL, response time, annotations, and more. Incorrect filter syntax or time ranges cause empty results."
---

## Phase 1 — Triage

MUST:
- Verify the time range contains traces: `aws xray get-trace-summaries --start-time <start> --end-time <end>`
- Test the filter expression syntax: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression '<expression>'`
- Check if traces exist at all in the time range (without filters)
- Verify annotation keys used in filters match actual annotation names

SHOULD:
- Check if the 30-day retention limit affects the query
- Verify service names in filter expressions match actual service names
- Test with simpler filter expressions to isolate the issue

MAY:
- Check X-Ray groups for pre-configured filter expressions
- Review trace annotation naming conventions

## Phase 2 — Remediate

MUST:
- Fix filter expression syntax (use X-Ray filter expression language)
- Adjust time range to cover the relevant period
- Use correct service names and annotation keys

SHOULD:
- Use annotations for searchable trace metadata (not metadata field)
- Create X-Ray groups for commonly used filter expressions

MAY:
- Export trace data for external analysis
- Set up CloudWatch dashboards with X-Ray metrics

## Common Issues

- symptoms: "Filter expression returns no results"
  diagnosis: "Incorrect filter syntax, wrong service name, or time range issue."
  resolution: "Test without filters first. Then add filters incrementally."

- symptoms: "Cannot filter by custom attribute"
  diagnosis: "Attribute stored as metadata instead of annotation."
  resolution: "Annotations are searchable; metadata is not. Use annotations for filterable data."

- symptoms: "Traces from last month not found"
  diagnosis: "Traces older than 30 days are deleted."
  resolution: "X-Ray retains traces for 30 days only. Check exported data or CloudWatch metrics."

## Output Format

```yaml
root_cause: "trace_analysis_issue — <specific_cause>"
evidence:
  - type: filter_expression
    content: "<filter expression used>"
  - type: trace_count
    content: "<traces found with and without filter>"
  - type: time_range
    content: "<query time range>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter expression or time range"
  long_term: "Standardize annotation naming and create X-Ray groups"
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
