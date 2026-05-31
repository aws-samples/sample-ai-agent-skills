---
title: "D2 — User Journey Analysis"
description: "Diagnose user journey tracking and analysis issues"
status: active
severity: MEDIUM
triggers: ["user journey not tracked", "page navigation missing", "session flow incomplete", "journey analysis error"]
owner: devops-agent
objective: "Resolve user journey tracking and analysis issues"
context: "User journey analysis tracks page navigations within a session. It requires HTTP and performance telemetry enabled and sufficient session data."
---

## Phase 1 — Triage

MUST:
- Verify telemetry includes HTTP and performance: `aws rum get-app-monitor --name <name> --query 'AppMonitor.AppMonitorConfiguration.Telemetries'`
- Check for session data: `aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 10`
- Verify the app monitor is collecting navigation events

SHOULD:
- Check if the application is a Single Page Application (SPA) — SPAs require additional configuration for route change tracking
- Verify the session sample rate provides sufficient journey data
- Check if the JavaScript snippet is configured to track page views

MAY:
- Compare journey data with server-side analytics
- Check if specific page transitions are missing from the data
- Verify the RUM web client version supports SPA route tracking

## Phase 2 — Remediate

MUST:
- Enable HTTP and performance telemetry if not configured
- For SPAs: configure the RUM web client to track route changes
- Ensure session sample rate is sufficient for journey analysis

SHOULD:
- Configure page groups for meaningful journey categorization
- Verify the snippet tracks all relevant page transitions

MAY:
- Set up custom events for key user journey milestones
- Configure CloudWatch metric export for journey-related metrics

## Common Issues

- symptoms: "Only the initial page load is tracked, subsequent navigations are missing"
  diagnosis: "Application is likely a SPA and the RUM client is not configured to track client-side route changes."
  resolution: "Configure the RUM web client for SPA support. Use the routeChangeComplete and routeChangeStart APIs to track client-side navigation."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
  - "update-app-monitor telemetries: YELLOW — changes data collection scope"
  - "RUM client SPA configuration: YELLOW — modifies client-side tracking behavior"
```

## Escalation Conditions

- Journey data consistently incomplete despite correct configuration
- SPA route tracking causing application errors
- Journey analysis needed for critical business decisions but data is insufficient
- Page group configuration not matching actual application structure

## Data Sensitivity

- `aws rum get-app-monitor-data`: MEDIUM — contains page URLs and navigation patterns
- User journey data: MEDIUM — reveals user behavior patterns and page visit sequences

## Prohibited Actions

- NEVER modify application routing to fix RUM journey tracking
- NEVER assume all page transitions are captured in SPAs without SPA-specific configuration
- NEVER share user journey data without authorization
- NEVER use journey data to identify individual users without proper consent

## Phase 3 — Rollback

1. If telemetry configuration was changed, revert to previous settings
2. If SPA tracking code was added, revert application changes if causing issues
3. Verify standard data collection is unaffected
4. Document the journey analysis issue and resolution
5. Confirm no application routing was affected by troubleshooting

## Output Format

```yaml
root_cause: "user_journey — <specific_cause>"
evidence:
  - type: journey_issue
    content: "<journey tracking configuration and data details>"
severity: MEDIUM
mitigation:
  immediate: "Fix journey tracking configuration"
  long_term: "Implement comprehensive SPA tracking and page group configuration"
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
