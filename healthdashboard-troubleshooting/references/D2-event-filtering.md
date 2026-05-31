---
title: "D2 — Event Filtering"
description: "Diagnose Health API event filtering issues"
status: active
severity: LOW
triggers:
  - "event filter not working"
  - "wrong events returned"
  - "filter syntax error"
  - "missing events in results"
owner: devops-agent
objective: "Resolve Health API event filtering issues"
context: "Health API supports filtering by event type categories, services, regions, availability zones, event status codes, and time ranges. Filters use specific syntax. Events older than 90 days are not returned regardless of filter."
---

## Phase 1 — Triage

MUST:
- Verify filter syntax: `aws health describe-events --filter '{"eventTypeCategories":["issue"],"eventStatusCodes":["open"]}' --region us-east-1`
- Check if events exist in the specified time range
- Verify filter values are valid (correct category names, status codes)
- Confirm events are within 90-day retention window

SHOULD:
- Try broader filter to confirm events exist
- Check filter field names match API specification
- Verify region filter uses correct region codes

MAY:
- Test with no filter to see all available events
- Check API pagination for large result sets

## Phase 2 — Remediate

MUST:
- Use correct filter field names: eventTypeCategories, services, regions, availabilityZones, eventStatusCodes, startTimes, endTimes
- Use valid values: categories are issue, accountNotification, scheduledChange
- Status codes are upcoming, open, closed
- Ensure time range is within 90-day retention

SHOULD:
- Start with broad filters and narrow down
- Use pagination for large result sets: `--max-results` and `--next-token`
- Combine filters for precise results

MAY:
- Implement filter presets for common queries
- Cache filter results for dashboard display

## Common Issues

- symptoms: "Filter returns no results"
  diagnosis: "No events match the filter criteria, or events are older than 90 days."
  resolution: "Broaden filter. Check time range. Verify events exist with no filter."

- symptoms: "Filter syntax error"
  diagnosis: "Invalid filter field name or value format."
  resolution: "Use correct field names and JSON format. Check API documentation."

- symptoms: "Missing events in filtered results"
  diagnosis: "Filter too restrictive or pagination not handled."
  resolution: "Broaden filter. Handle pagination with --next-token."

## Output Format

```yaml
root_cause: "event_filtering — <specific_cause>"
evidence:
  - type: filter_config
    content: "<filter parameters used>"
  - type: results
    content: "<events returned or error message>"
severity: LOW
mitigation:
  immediate: "Fix filter syntax or broaden criteria"
  long_term: "Implement reusable filter presets and pagination handling"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
