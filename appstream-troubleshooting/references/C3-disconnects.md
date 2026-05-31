---
title: "C3 — Session Disconnects"
description: "Diagnose AppStream 2.0 frequent session disconnects"
status: active
severity: MEDIUM
triggers:
  - "session disconnecting"
  - "frequent disconnects"
  - "session dropped"
  - "connection lost"
owner: devops-agent
objective: "Identify and resolve frequent session disconnect issues"
context: "Disconnects can be caused by network instability, idle timeout settings, max session duration, proxy interference, or fleet scaling events. Disconnected sessions can be preserved for reconnection based on the disconnect timeout setting."
---

## Phase 1 — Triage

MUST:
- Check fleet timeout settings: `aws appstream describe-fleets --names <fleet-name> --query 'Fleets[*].{IdleDisconnect:IdleDisconnectTimeoutInSeconds,Disconnect:DisconnectTimeoutInSeconds,MaxSession:MaxUserDurationInSeconds}'`
- Check if disconnects correlate with idle periods
- Verify network stability between user and AppStream
- Check CloudWatch for session metrics

SHOULD:
- Verify proxy/firewall is not terminating WebSocket connections
- Check if the AppStream client or browser is up to date
- Verify the fleet is not scaling down during active sessions

MAY:
- Check if specific users or locations experience more disconnects
- Review network logs for connection drops

## Phase 2 — Remediate

MUST:
- Adjust idle disconnect timeout if users are being disconnected too quickly:
  `aws appstream update-fleet --name <fleet-name> --idle-disconnect-timeout-in-seconds <seconds>`
- Adjust disconnect timeout for session preservation:
  `aws appstream update-fleet --name <fleet-name> --disconnect-timeout-in-seconds <seconds>`
- Fix network issues (proxy, firewall, bandwidth)

SHOULD:
- Configure appropriate max session duration
- Ensure proxies support long-lived WebSocket connections
- Use the AppStream client for more stable connections

MAY:
- Set up monitoring for disconnect frequency
- Configure session reconnection settings

## Common Issues

- symptoms: "Users disconnected after being idle"
  diagnosis: "Idle disconnect timeout too short."
  resolution: "Increase IdleDisconnectTimeoutInSeconds (5-360 minutes)."

- symptoms: "Disconnected session cannot be reconnected"
  diagnosis: "Disconnect timeout expired — session terminated."
  resolution: "Increase DisconnectTimeoutInSeconds to allow more time for reconnection."

- symptoms: "Random disconnects during active use"
  diagnosis: "Network instability or proxy terminating WebSocket connections."
  resolution: "Check network stability. Configure proxy to allow long-lived WebSocket connections."

## Output Format

```yaml
root_cause: "session_disconnect — <specific_cause>"
evidence:
  - type: timeout_settings
    content: "<fleet timeout configuration>"
  - type: disconnect_pattern
    content: "<disconnect frequency and timing>"
  - type: network_status
    content: "<network connectivity status>"
severity: MEDIUM
mitigation:
  immediate: "Adjust timeout settings or fix network issues"
  long_term: "Monitor disconnect rates and optimize network path"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Adjust idle disconnect timeout | YELLOW | Fleet configuration change |
| Adjust disconnect timeout | YELLOW | Fleet configuration change |
| Fix network issues | YELLOW | Network/proxy changes |
| Configure max session duration | YELLOW | Fleet configuration change |
| Ensure proxies support WebSocket | YELLOW | Proxy configuration change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Timeout settings**: Fleet disconnect configuration — INTERNAL
- **Session data**: User session state — CONFIDENTIAL
- **Network status**: Connectivity and proxy details — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER set disconnect timeout to 0 (immediate termination)
- NEVER modify timeout settings during active business hours without notice

## Phase 3 — Rollback

1. Revert idle disconnect timeout to previous value
2. Revert disconnect timeout to previous value
3. Revert max session duration to previous value
4. Restore previous proxy/network configuration
5. Verify active sessions are not disrupted by the rollback

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
