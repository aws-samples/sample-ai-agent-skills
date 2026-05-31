---
title: "B4 — Pipe Filtering Issues"
description: "Diagnose event filtering problems in EventBridge Pipes"
status: active
severity: MEDIUM
triggers:
  - "filter not matching"
  - "events filtered out"
  - "filter pattern"
  - "no events passing filter"
  - "wrong events"
owner: devops-agent
objective: "Identify and resolve event filtering issues in Pipes"
context: "Pipe filtering uses event patterns to select which source events are processed. Filters are applied before enrichment and target delivery. Filter patterns use the same syntax as EventBridge event patterns but are applied to the source event format."
---

## Phase 1 — Triage

MUST:
- Describe the pipe for filter configuration: `aws pipes describe-pipe --name <pipe-name>`
- Review the FilterCriteria in the pipe configuration
- Verify the filter pattern syntax matches the source event format
- Check if events exist in the source that match the filter

SHOULD:
- Compare the filter pattern against actual source event structure
- Check for case sensitivity issues in filter values
- Verify filter uses correct JSON path for the source type
- Test the filter pattern with sample events

MAY:
- Check pipe execution metrics for filtered vs processed events
- Review source event format documentation for the specific source type
- Compare with EventBridge event pattern documentation

## Phase 2 — Remediate

MUST:
- Fix the filter pattern to match the actual source event structure
- Verify the filter pattern syntax is valid JSON
- Test with actual source events

SHOULD:
- Start with a broad filter and narrow down
- Document the expected event structure and filter logic
- Use the EventBridge pattern tester for validation

MAY:
- Implement filter pattern versioning
- Set up monitoring for filter match rates

## Common Issues

- symptoms: "No events passing the filter despite source having events"
  diagnosis: "Filter pattern does not match the source event structure (different field names or nesting)."
  resolution: "Check the actual source event format. SQS body, Kinesis data, DynamoDB record structures differ."

- symptoms: "Too many events passing the filter"
  diagnosis: "Filter pattern is too broad or missing required fields."
  resolution: "Add more specific filter criteria. Use content-based filtering with exact values."

- symptoms: "Filter works for some events but not others"
  diagnosis: "Event structure varies and some events don't have the filtered field."
  resolution: "Use exists filter for optional fields. Handle event structure variations in the pattern."

## Output Format

```yaml
root_cause: "filtering — <specific_cause>"
evidence:
  - type: pipe
    content: "<pipe filter configuration>"
  - type: source_events
    content: "<sample source event structure>"
  - type: filter_analysis
    content: "<filter pattern vs event structure comparison>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter pattern to match source event structure"
  long_term: "Document event structures and test filters before deployment"
```


## Safety Ratings
```
safety_ratings:
  - "Describe pipe filter configuration: GREEN — read-only API call"
  - "Review FilterCriteria: GREEN — read-only analysis"
  - "Compare filter against source events: GREEN — read-only analysis"
  - "Update filter pattern: YELLOW — changes which events are processed, may increase or decrease volume"
  - "Remove filter entirely: YELLOW — all source events will be processed, potential volume spike"
```

## Escalation Conditions
- Pipe serves production event processing pipeline
- Filter blocking all events causing processing gap
- Filter too broad causing target overload
- Filter change needed during active production traffic
- Filter logic requires understanding of source event schema changes

## Data Sensitivity
```
data_sensitivity:
  classification: LOW
  sensitive_fields:
    - "Filter patterns: business logic for event selection"
    - "Source event structure: data schema information"
  handling: "Filter patterns may reveal business logic. Do not expose externally."
```

## Prohibited Actions
- NEVER remove all filters from a high-volume pipe without understanding the throughput impact on the target
- NEVER modify filter patterns in production without testing against sample events first
- NEVER assume source event structure is static — verify against actual events

## Phase 3 — Rollback
- If filter pattern was changed: restore previous FilterCriteria configuration
- If filter was removed: re-add the original filter pattern
- If filter was made more restrictive: broaden back to previous pattern
- If filter was made more permissive: restore previous restrictive pattern to reduce volume

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
