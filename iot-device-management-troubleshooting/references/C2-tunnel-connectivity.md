---
title: "C2 — Tunnel Connectivity"
description: "Diagnose tunnel connection drops, proxy failures, and data transfer issues"
status: active
severity: MEDIUM
triggers: ["tunnel disconnected", "proxy connection failed", "tunnel data not flowing", "WebSocket error"]
owner: devops-agent
objective: "Resolve tunnel connectivity issues and restore remote device access"
context: "Secure tunnels use WebSocket connections. Both source and destination require local proxy applications. Connection issues can be network, proxy, or token related."
---

## Phase 1 — Triage

MUST:
- Check tunnel status: `aws iotsecuretunneling describe-tunnel --tunnel-id <tunnel-id> --query '{Status:tunnel.status,SourceConnectionState:tunnel.sourceConnectionState,DestinationConnectionState:tunnel.destinationConnectionState}'`
- Verify tunnel has not expired: check createdAt vs timeoutConfig
- Confirm both source and destination connection states

SHOULD:
- Check if local proxy process is running on source machine
- Verify destination device connectivity: `aws iot search-index --query-string "thingName:<name> AND connectivity.connected:true"`
- Test WebSocket connectivity to tunneling endpoint from source

MAY:
- Review local proxy logs for connection errors
- Check network firewall rules for WebSocket (wss://) traffic on port 443
- Verify proxy version compatibility with tunneling service

## Phase 2 — Remediate

MUST:
- Restart local proxy if it crashed or lost connection
- If tunnel expired, open a new tunnel: `aws iotsecuretunneling open-tunnel --destination-config thingName=<name>,services=SSH`
- Ensure network path allows outbound WebSocket connections on port 443

SHOULD:
- Update local proxy to latest version for bug fixes
- Configure proxy with appropriate connection timeout and retry settings
- Verify destination service (e.g., sshd) is running on the device

MAY:
- Use rotate-tunnel-access-token if token may be corrupted: `aws iotsecuretunneling rotate-tunnel-access-token --tunnel-id <id> --client-mode DESTINATION`
- Set up monitoring for tunnel connection state changes
- Configure proxy keepalive settings to prevent idle disconnections

## Safety Ratings

```
safety_ratings:
  - "describe-tunnel: GREEN — read-only inspection"
  - "search-index: GREEN — read-only fleet query"
  - "open-tunnel: YELLOW — creates new tunnel with access tokens"
  - "rotate-tunnel-access-token: YELLOW — invalidates current token, issues new one"
  - "close-tunnel: YELLOW — terminates active session"
```

## Escalation Conditions

- Persistent connectivity failures across multiple devices indicating network infrastructure issue
- Local proxy crashes repeatedly suggesting compatibility or resource issues
- Tunnel connections drop during critical maintenance operations
- Corporate firewall blocking WebSocket traffic to AWS tunneling endpoints

## Data Sensitivity

- `aws iotsecuretunneling describe-tunnel`: LOW — returns connection state metadata
- Local proxy logs: MEDIUM — may contain connection details and error traces
- Network diagnostic output: LOW — contains connectivity test results
- `aws iot search-index`: MEDIUM — returns device connectivity information

## Prohibited Actions

- NEVER bypass network security controls to establish tunnel connectivity
- NEVER run local proxy with elevated privileges unless absolutely required
- NEVER ignore persistent connection drops — they may indicate security issues
- NEVER use tunnel connectivity for bulk data transfer — tunnels are for interactive sessions

## Phase 3 — Rollback

1. Close problematic tunnel: `aws iotsecuretunneling close-tunnel --tunnel-id <tunnel-id>`
2. Open a fresh tunnel with correct configuration
3. If proxy was updated, revert to previous version if new version causes issues
4. Restore firewall rules if they were modified for troubleshooting
5. Verify connectivity is stable before resuming maintenance operations

## Common Issues

- symptoms: "Source proxy connects but destination never establishes connection"
  diagnosis: "Destination local proxy is not running, or device did not receive the destination access token."
  resolution: "Verify destination proxy is running and was started with the correct destination token. Check MQTT notification delivery."

- symptoms: "Tunnel shows OPEN but SSH connection times out"
  diagnosis: "Local proxy is connected but the destination service (sshd) is not running or not listening on the expected port."
  resolution: "Verify sshd is running on the device. Ensure destination proxy is configured to forward to the correct local port."

## Output Format

```yaml
root_cause: "tunnel_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Restore tunnel connectivity"
  long_term: "Implement proxy health monitoring and auto-restart"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
