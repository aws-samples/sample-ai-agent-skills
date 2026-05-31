---
title: "C1 — Custom Event Errors"
description: "Diagnose custom event configuration and collection issues"
status: active
severity: MEDIUM
triggers: ["custom events not working", "recordEvent failed", "custom event missing", "event not collected"]
owner: devops-agent
objective: "Resolve custom event configuration and collection issues"
context: "Custom events require explicit enablement in the app monitor and application code to call the RUM web client recordEvent API."
---

## Phase 1 — Triage

MUST:
- Check if custom events are enabled: `aws rum get-app-monitor --name <name> --query 'AppMonitor.CustomEvents'`
- Verify the app monitor is ACTIVE and collecting standard events
- Check if the application code calls the RUM web client recordEvent API

SHOULD:
- Verify the custom event name and payload structure
- Check browser console for errors when recording custom events
- Verify the RUM web client is initialized before recordEvent is called

MAY:
- Check data delivery destination for custom event data
- Test with a simple custom event to isolate the issue
- Verify the custom event data appears in get-app-monitor-data results

## Phase 2 — Remediate

MUST:
- Enable custom events if disabled: `aws rum update-app-monitor --name <name> --custom-events '{"Status":"ENABLED"}'`
- Ensure the application calls recordEvent after the RUM client is initialized
- Verify the event name and payload are valid

SHOULD:
- Implement error handling around recordEvent calls
- Test custom events in a development environment first
- Document the expected custom event schema

MAY:
- Set up monitoring for custom event collection rates
- Configure metric definitions based on custom events

## Common Issues

- symptoms: "recordEvent called but custom events not appearing in RUM data"
  diagnosis: "Custom events may not be enabled in the app monitor configuration."
  resolution: "Enable custom events: `aws rum update-app-monitor --name <name> --custom-events '{\"Status\":\"ENABLED\"}'`. Verify the RUM client is initialized before calling recordEvent."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "update-app-monitor custom-events: YELLOW — enables additional data collection"
  - "recordEvent (client-side): GREEN — sends event data from browser"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
```

## Escalation Conditions

- Custom events enabled but consistently not appearing in data
- Custom event collection causing performance issues in the application
- Custom event payload size exceeding limits
- Custom events needed for critical business metrics but not collecting

## Data Sensitivity

- `aws rum get-app-monitor`: MEDIUM — reveals monitoring configuration
- Custom event data: MEDIUM to HIGH — may contain business-specific user interaction data
- `aws rum get-app-monitor-data`: MEDIUM — contains user session and event data

## Prohibited Actions

- NEVER include PII or sensitive data in custom event payloads
- NEVER enable custom events without understanding the data volume impact
- NEVER modify the RUM client initialization in production without testing
- NEVER assume custom events are collected if the feature is not explicitly enabled

## Phase 3 — Rollback

1. If custom events were enabled in error, disable them: `aws rum update-app-monitor --name <name> --custom-events '{"Status":"DISABLED"}'`
2. If application code was modified, revert the recordEvent changes
3. Verify standard data collection is unaffected by custom event changes
4. Document the custom event configuration and any code changes
5. Confirm no sensitive data was inadvertently collected

## Output Format

```yaml
root_cause: "custom_events — <specific_cause>"
evidence:
  - type: event_error
    content: "<custom event configuration and error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix custom event configuration or application code"
  long_term: "Implement custom event testing in CI/CD pipeline"
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
