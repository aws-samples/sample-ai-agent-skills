---
title: "C2 — Page Performance Analysis"
description: "Diagnose page performance data issues and analysis problems"
status: active
severity: MEDIUM
triggers: ["performance data missing", "page load times wrong", "Web Vitals not showing", "performance analysis issue"]
owner: devops-agent
objective: "Resolve page performance data collection and analysis issues"
context: "RUM collects Web Vitals (LCP, FID, CLS) and navigation timing data. Performance telemetry must be enabled. Data depends on browser API support."
---

## Phase 1 — Triage

MUST:
- Verify performance telemetry is enabled: `aws rum get-app-monitor --name <name> --query 'AppMonitor.AppMonitorConfiguration.Telemetries'`
- Check for performance data: `aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 10`
- Verify the app monitor is ACTIVE and collecting data

SHOULD:
- Check if the session sample rate is sufficient for meaningful performance data
- Verify the JavaScript snippet is loaded in the `<head>` before other scripts
- Check browser compatibility for Web Vitals APIs

MAY:
- Compare RUM performance data with other monitoring tools (Lighthouse, PageSpeed)
- Check if specific pages have different performance characteristics
- Verify no JavaScript errors are preventing performance data collection

## Phase 2 — Remediate

MUST:
- Enable performance telemetry if disabled: include "performance" in Telemetries array
- Ensure the snippet loads early in the page for accurate timing
- Verify the app monitor has sufficient data (check session sample rate)

SHOULD:
- Increase session sample rate if performance data is sparse
- Verify the snippet is not deferred or async-loaded (affects timing accuracy)
- Check that the application does not override browser performance APIs

MAY:
- Configure CloudWatch metric export for performance metrics
- Set up alarms on key Web Vitals thresholds

## Common Issues

- symptoms: "LCP and FID metrics showing but CLS is always 0"
  diagnosis: "CLS (Cumulative Layout Shift) may genuinely be 0 if the page has no layout shifts, or the browser may not support the Layout Instability API."
  resolution: "Verify CLS in browser dev tools (Performance tab). Check browser compatibility. CLS of 0 may be correct for well-optimized pages."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
  - "update-app-monitor telemetries: YELLOW — changes data collection scope"
  - "cloudwatch put-metric-alarm: YELLOW — creates new alarm resource"
```

## Escalation Conditions

- Performance data consistently missing despite correct configuration
- Performance metrics show values that contradict other monitoring tools
- Web Vitals data needed for SLA reporting but not collecting
- Performance data collection causing noticeable page load impact

## Data Sensitivity

- `aws rum get-app-monitor-data`: MEDIUM — contains page URLs and performance metrics
- Performance data: LOW — contains timing metrics, not user content

## Prohibited Actions

- NEVER modify the application's JavaScript to fix RUM performance collection without testing
- NEVER dismiss performance data discrepancies without investigating the cause
- NEVER assume all browsers provide the same performance metrics
- NEVER use RUM performance data as the sole source for SLA reporting without validation

## Phase 3 — Rollback

1. If telemetry configuration was changed, revert to previous settings
2. If the snippet loading order was modified, restore original placement
3. Verify performance data collection resumes after changes
4. Document the performance analysis issue and resolution
5. Confirm no application performance was affected by troubleshooting changes

## Output Format

```yaml
root_cause: "page_performance — <specific_cause>"
evidence:
  - type: performance_issue
    content: "<performance data and configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix performance data collection"
  long_term: "Set up automated Web Vitals monitoring and alerting"
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
