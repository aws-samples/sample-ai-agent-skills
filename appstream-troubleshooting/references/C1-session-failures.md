---
title: "C1 — Streaming Session Failures"
description: "Diagnose AppStream 2.0 streaming session failures"
status: active
severity: HIGH
triggers:
  - "session failed"
  - "cannot start session"
  - "streaming error"
  - "session not connecting"
owner: devops-agent
objective: "Identify and resolve streaming session failures"
context: "Session failures can be caused by insufficient fleet capacity, network issues, authentication problems, browser compatibility, or fleet configuration errors. Users connect via web browser or AppStream client."
---

## Phase 1 — Triage

MUST:
- Check fleet status and capacity: `aws appstream describe-fleets --names <fleet-name>`
- Check for InsufficientCapacityError: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name InsufficientCapacityError --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify the stack-fleet association: `aws appstream describe-stacks --names <stack-name>`
- Check active sessions: `aws appstream describe-sessions --stack-name <stack-name> --fleet-name <fleet-name>`

SHOULD:
- Verify the user is authenticated and authorized
- Check if the streaming URL is valid and not expired
- Verify network connectivity to AppStream streaming endpoints
- Check browser compatibility (Chrome, Firefox, Edge, Safari)

MAY:
- Check CloudTrail for session-related events
- Verify the AppStream client version if using the native client

## Phase 2 — Remediate

MUST:
- Ensure fleet has available capacity (increase minimum capacity if needed)
- Verify stack-fleet association is active
- Fix authentication configuration

SHOULD:
- Generate a valid streaming URL: `aws appstream create-streaming-url --stack-name <stack-name> --fleet-name <fleet-name> --user-id <user-id>`
- Ensure HTTPS (443) and UDP 8433 are allowed through firewalls
- Use a supported browser

MAY:
- Increase fleet capacity for peak usage
- Set up monitoring for session failures

## Common Issues

- symptoms: "User gets 'no available instances' error"
  diagnosis: "Fleet has no available capacity."
  resolution: "Increase fleet minimum capacity or wait for scaling to add instances."

- symptoms: "Streaming URL returns error"
  diagnosis: "URL expired or stack-fleet association broken."
  resolution: "Generate a new streaming URL. Verify stack-fleet association."

- symptoms: "Session connects but immediately disconnects"
  diagnosis: "Network issue or firewall blocking streaming traffic."
  resolution: "Allow HTTPS (443) and UDP 8433 to AppStream endpoints."

## Output Format

```yaml
root_cause: "session_failure — <specific_cause>"
evidence:
  - type: fleet_capacity
    content: "<fleet capacity status>"
  - type: session_status
    content: "<session state>"
  - type: network_connectivity
    content: "<network access to streaming endpoints>"
severity: HIGH
mitigation:
  immediate: "Fix the session failure cause"
  long_term: "Monitor capacity and set up alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure fleet has available capacity | YELLOW | May require scaling changes |
| Verify stack-fleet association | GREEN | Configuration check |
| Fix authentication configuration | YELLOW | Auth configuration change |
| Generate valid streaming URL | GREEN | URL generation |
| Allow HTTPS and UDP 8433 | YELLOW | Firewall/network change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Session data**: User session state and activity — CONFIDENTIAL
- **Streaming URLs**: Temporary access URLs — CONFIDENTIAL
- **Authentication config**: User pool or SAML settings — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER share streaming URLs with unauthorized users
- NEVER terminate active sessions without user notification

## Phase 3 — Rollback

1. Revert fleet capacity changes if scaling was modified
2. Restore previous stack-fleet association if it was changed
3. Revert authentication configuration changes
4. Revert firewall/network changes if they were made
5. Verify existing sessions are not disrupted

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
