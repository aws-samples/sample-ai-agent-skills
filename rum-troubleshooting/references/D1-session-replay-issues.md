---
title: "D1 — Session Replay Issues"
description: "Diagnose session replay configuration and playback problems"
status: active
severity: MEDIUM
triggers: ["session replay not working", "no replay data", "replay playback error", "session recording missing"]
owner: devops-agent
objective: "Resolve session replay configuration and data issues"
context: "Session replay captures DOM changes and user interactions. It must be explicitly enabled and significantly increases data volume and cost."
---

## Phase 1 — Triage

MUST:
- Check if session replay is enabled: `aws rum get-app-monitor --name <name> --query 'AppMonitor.AppMonitorConfiguration'`
- Verify the app monitor is ACTIVE and collecting standard data
- Check if the session sample rate is greater than 0

SHOULD:
- Verify the RUM web client version supports session replay
- Check if the application's CSP allows session replay data transmission
- Verify the Cognito Identity pool IAM role has sufficient permissions

MAY:
- Check browser console for session replay-related errors
- Verify the application does not have elements that block DOM observation
- Check data volume to confirm replay data is being generated

## Phase 2 — Remediate

MUST:
- Enable session replay in app monitor configuration if disabled
- Ensure the RUM web client version supports session replay
- Verify CSP allows the necessary network connections

SHOULD:
- Configure session replay privacy settings (mask inputs, exclude elements)
- Set appropriate session sample rate considering replay cost impact
- Test session replay in a development environment first

MAY:
- Configure element masking for sensitive content
- Set up monitoring for replay data volume and cost

## Common Issues

- symptoms: "Session replay enabled but no replay data available"
  diagnosis: "RUM web client version may not support session replay, or CSP blocks replay data."
  resolution: "Verify RUM web client version is 1.12.0 or later. Check CSP allows connect-src for RUM endpoints. Verify session sample rate is > 0."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "update-app-monitor session replay: YELLOW — enables DOM recording, increases cost"
  - "Session replay data: RED — captures user interactions and page content"
```

## Escalation Conditions

- Session replay captures sensitive data that should be masked
- Replay data volume causing unexpected cost increases
- Session replay affecting application performance
- Privacy or compliance concerns about recorded session data

## Data Sensitivity

- `aws rum get-app-monitor`: MEDIUM — reveals monitoring configuration
- Session replay data: HIGH — captures DOM content, user interactions, and potentially sensitive page content
- Replay playback: HIGH — shows actual user sessions

## Prohibited Actions

- NEVER enable session replay without privacy review and input masking configuration
- NEVER enable session replay on pages with sensitive data (payment forms, health records) without proper masking
- NEVER share session replay data without authorization
- NEVER enable session replay with 100% sample rate on high-traffic applications without cost analysis

## Phase 3 — Rollback

1. Disable session replay if enabled in error: update app monitor configuration
2. If privacy-sensitive data was captured, assess the data retention and deletion options
3. Revert RUM web client version if the update caused issues
4. Verify standard data collection is unaffected
5. Document the session replay configuration change and any privacy implications

## Output Format

```yaml
root_cause: "session_replay — <specific_cause>"
evidence:
  - type: replay_issue
    content: "<session replay configuration and error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix session replay configuration"
  long_term: "Implement privacy-aware session replay with proper masking"
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
