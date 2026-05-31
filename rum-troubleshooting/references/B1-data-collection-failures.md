---
title: "B1 — Data Collection Failures"
description: "Diagnose RUM data collection failures and missing data"
status: active
severity: HIGH
triggers: ["no RUM data", "data not collecting", "missing data", "no events received"]
owner: devops-agent
objective: "Resolve data collection failures for CloudWatch RUM"
context: "Data collection requires: JavaScript snippet installed, Cognito Identity pool configured, correct domain, and real user traffic."
---

## Phase 1 — Triage

MUST:
- Verify app monitor is ACTIVE: `aws rum get-app-monitor --name <name>`
- Check for recent data: `aws rum get-app-monitor-data --name <name> --time-range '{"After":<epoch-start>,"Before":<epoch-end>}' --max-results 5`
- Verify Cognito Identity pool is configured and accessible: `aws cognito-identity describe-identity-pool --identity-pool-id <pool-id>`

SHOULD:
- Verify the JavaScript snippet is installed in the application (check page source)
- Check if the Cognito Identity pool allows unauthenticated identities
- Verify the IAM role associated with the identity pool has rum:PutRumEvents permission
- Check the session sample rate — a rate of 0 means no sessions are captured

MAY:
- Check browser developer console for RUM-related errors
- Verify Content Security Policy (CSP) allows RUM endpoints
- Check if the application domain matches the app monitor's domain configuration

## Phase 2 — Remediate

MUST:
- Install JavaScript snippet if missing
- Fix Cognito Identity pool to allow unauthenticated identities
- Grant rum:PutRumEvents permission to the Cognito unauthenticated role
- Ensure session sample rate is greater than 0

SHOULD:
- Verify the snippet loads before other scripts for accurate timing
- Update CSP headers to allow RUM and Cognito endpoints
- Verify the domain configuration matches the application domain

MAY:
- Increase session sample rate if data volume is too low
- Enable additional telemetry types for more comprehensive data

## Common Issues

- symptoms: "App monitor ACTIVE but no data appearing"
  diagnosis: "JavaScript snippet may not be installed, Cognito may be misconfigured, or CSP may block RUM."
  resolution: "Check in order: 1) Snippet installed in HTML, 2) Cognito Identity pool allows unauthenticated, 3) IAM role has rum:PutRumEvents, 4) CSP allows RUM endpoints, 5) Session sample rate > 0."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "get-app-monitor-data: GREEN — read-only data retrieval"
  - "describe-identity-pool: GREEN — read-only Cognito details"
  - "update-identity-pool: YELLOW — modifies authentication configuration"
```

## Escalation Conditions

- Data collection fails despite all configuration appearing correct
- Cognito Identity pool changes affect other applications using the same pool
- CSP changes required on a production application
- Data collection works intermittently suggesting a race condition

## Data Sensitivity

- `aws rum get-app-monitor-data`: MEDIUM — contains user session data and performance metrics
- `aws cognito-identity describe-identity-pool`: MEDIUM — reveals authentication configuration
- Browser console logs: MEDIUM — may contain user session information

## Prohibited Actions

- NEVER modify Cognito Identity pool settings without checking impact on other applications
- NEVER disable CSP to troubleshoot RUM data collection
- NEVER set session sample rate to 1.0 on high-traffic sites without cost analysis
- NEVER assume data collection failure means the application is broken

## Phase 3 — Rollback

1. If Cognito Identity pool was modified, revert to previous configuration
2. If CSP headers were changed, restore original CSP policy
3. If session sample rate was changed, revert to previous value
4. Verify data collection resumes after fixes
5. Document the data collection issue and resolution

## Output Format

```yaml
root_cause: "data_collection — <specific_cause>"
evidence:
  - type: collection_failure
    content: "<collection configuration and error details>"
severity: HIGH
mitigation:
  immediate: "Fix data collection pipeline"
  long_term: "Implement data collection health monitoring"
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
