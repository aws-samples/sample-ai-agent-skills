---
title: "E1 — Bot Integration"
description: "Diagnose Wickr bot integration failures"
status: active
severity: MEDIUM
triggers: ["bot not responding", "bot connection failed", "bot SDK error", "bot offline"]
owner: devops-agent
objective: "Resolve Wickr bot integration and connectivity issues"
context: "Wickr bots use the Bot SDK and run as long-lived processes on compute infrastructure. They are not Lambda functions."
---

## Phase 1 — Triage

MUST:
- Verify bot compute instance is running (EC2/ECS/EKS)
- Check bot process logs for connection errors
- Verify network is active: `aws wickr get-network --network-id <id>`

SHOULD:
- Check bot credentials and API token validity
- Verify network connectivity from bot host to Wickr service endpoints
- Review CloudWatch metrics for bot activity: `aws cloudwatch get-metric-statistics --namespace AWS/Wickr --metric-name BotMessages --dimensions Name=NetworkId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

MAY:
- Test bot connectivity with a simple ping/echo command
- Check bot SDK version for known issues

## Phase 2 — Remediate

MUST:
- Restart bot process if it has crashed or disconnected
- Regenerate bot API token if expired or revoked
- Ensure bot host has network access to Wickr service endpoints

SHOULD:
- Update bot SDK to latest version
- Implement bot health checks and automatic restart
- Configure bot logging for troubleshooting

MAY:
- Deploy bot on ECS with auto-restart capability
- Implement bot monitoring with CloudWatch custom metrics

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "bot process restart: YELLOW — temporary bot unavailability"
  - "bot token regeneration: YELLOW — invalidates existing token"
```

## Escalation Conditions

- Bot cannot connect to Wickr service after token regeneration
- Bot SDK throws unrecoverable errors
- Bot host infrastructure is failing (EC2/ECS issues)
- Bot integration required for critical business workflow

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- Bot API tokens: HIGH — authentication credentials
- Bot process logs: MEDIUM — may contain message metadata
- Bot configuration: MEDIUM — contains connection parameters

## Prohibited Actions

- NEVER share bot API tokens in logs or outputs
- NEVER run bots as Lambda functions (requires persistent connections)
- NEVER give bots admin-level network permissions
- NEVER deploy bots without proper authentication token management

## Phase 3 — Rollback

1. If bot token was regenerated, update all bot instances with new token
2. If bot configuration was changed, restore original configuration
3. Verify bot reconnects and processes messages after changes
4. Confirm bot-dependent workflows are functioning
5. Document bot integration changes

## Common Issues

- symptoms: "Bot disconnects frequently and requires manual restart"
  diagnosis: "Bot process not handling WebSocket reconnection properly."
  resolution: "Update bot SDK. Implement reconnection logic. Deploy on ECS with restart policy."

- symptoms: "Bot connects but does not receive messages"
  diagnosis: "Bot user account may not be in the correct rooms/channels."
  resolution: "Verify bot user is added to the appropriate rooms. Check bot permissions in admin console."

## Output Format

```yaml
root_cause: "bot_integration — <specific_cause>"
evidence:
  - type: bot_status
    content: "<status details>"
severity: MEDIUM
mitigation:
  immediate: "Restore bot connectivity"
  long_term: "Implement bot health monitoring and auto-restart"
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
