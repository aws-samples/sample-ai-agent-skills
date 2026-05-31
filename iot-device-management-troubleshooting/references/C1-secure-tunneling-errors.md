---
title: "C1 — Secure Tunneling Errors"
description: "Diagnose secure tunnel creation and token delivery failures"
status: active
severity: HIGH
triggers: ["tunnel creation failed", "tunnel token error", "secure tunneling not working", "cannot open tunnel"]
owner: devops-agent
objective: "Resolve secure tunnel creation failures and token delivery issues"
context: "Secure tunneling creates WebSocket tunnels for remote device access. Requires local proxy on both ends. Tokens are single-use and time-limited."
---

## Phase 1 — Triage

MUST:
- List recent tunnels: `aws iotsecuretunneling list-tunnels --query 'tunnelSummaries[*].{Id:tunnelId,Status:status,Description:description}'`
- Describe the failing tunnel: `aws iotsecuretunneling describe-tunnel --tunnel-id <tunnel-id>`
- Check tunnel status and token expiry: `aws iotsecuretunneling describe-tunnel --tunnel-id <tunnel-id> --query '{Status:tunnel.status,CreatedAt:tunnel.createdAt,TimeoutConfig:tunnel.timeoutConfig}'`

SHOULD:
- Verify IAM permissions for iotsecuretunneling:OpenTunnel
- Check if the target thing exists: `aws iot describe-thing --thing-name <name>`
- Verify tunnel destination configuration includes correct service (e.g., SSH)

MAY:
- Check CloudTrail for OpenTunnel events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=OpenTunnel --max-results 10`
- Review IoT device policy for iot:OpenTunnel permission on the thing
- Check regional availability of secure tunneling service

## Phase 2 — Remediate

MUST:
- Ensure IAM caller has iotsecuretunneling:OpenTunnel permission
- Verify thing-level authorization: IoT policy must allow iot:OpenTunnel for the target thing
- Ensure tunnel timeout is set appropriately (max 12 hours)

SHOULD:
- Verify destination device has the local proxy agent installed and running
- Ensure device can receive tunnel notification via MQTT topic $aws/things/<name>/tunnels/notify
- Check network allows WebSocket connections to tunneling endpoint

MAY:
- Tag tunnels for tracking: `aws iotsecuretunneling tag-resource --resource-arn <arn> --tags Key=Purpose,Value=Debugging`
- Configure tunnel timeout based on expected session duration

## Safety Ratings

```
safety_ratings:
  - "list-tunnels: GREEN — read-only enumeration"
  - "describe-tunnel: GREEN — read-only inspection"
  - "open-tunnel: YELLOW — creates tunnel with access tokens, tokens grant device access"
  - "close-tunnel: YELLOW — terminates active tunnel session"
  - "rotate-tunnel-access-token: YELLOW — invalidates existing token and issues new one"
```

## Escalation Conditions

- Tunnel tokens are being intercepted or misused (security incident)
- Tunneling service returns persistent 5xx errors indicating service issue
- Device fleet cannot receive tunnel notifications due to MQTT connectivity issues
- Compliance requires audit of all tunnel sessions and access patterns

## Data Sensitivity

- `aws iotsecuretunneling describe-tunnel`: LOW — returns tunnel metadata
- `aws iotsecuretunneling open-tunnel`: HIGH — returns access tokens that grant device access
- Tunnel access tokens: CRITICAL — must be transmitted securely and never logged
- `aws cloudtrail lookup-events`: MEDIUM — reveals who opened tunnels to which devices

## Prohibited Actions

- NEVER log or store tunnel access tokens in plain text
- NEVER share source and destination tokens through the same channel
- NEVER leave tunnels open longer than necessary — close after use
- NEVER open tunnels to production devices without authorization and audit trail

## Phase 3 — Rollback

1. Close the tunnel if it was opened in error: `aws iotsecuretunneling close-tunnel --tunnel-id <tunnel-id>`
2. If tokens were exposed, close tunnel immediately — tokens become invalid when tunnel is closed
3. Verify no unauthorized access occurred during the tunnel session
4. Review CloudTrail for any actions taken through the tunnel
5. Document the incident and update access policies if needed

## Common Issues

- symptoms: "OpenTunnel returns AccessDeniedException"
  diagnosis: "IAM policy missing iotsecuretunneling:OpenTunnel or IoT policy missing iot:OpenTunnel for the thing."
  resolution: "Add required permissions to IAM policy and IoT thing policy."

- symptoms: "Tunnel created but destination device never connects"
  diagnosis: "Device local proxy is not running or device did not receive tunnel notification via MQTT."
  resolution: "Verify device subscribes to $aws/things/<name>/tunnels/notify. Ensure local proxy is installed and configured."

## Output Format

```yaml
root_cause: "secure_tunneling — <specific_cause>"
evidence:
  - type: tunnel_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix tunnel configuration"
  long_term: "Automate local proxy deployment and tunnel monitoring"
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
