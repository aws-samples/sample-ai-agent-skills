---
title: "C1 — Message Delivery Failures"
description: "Diagnose message delivery failures"
status: active
severity: HIGH
triggers: ["message not delivered", "message failed", "delivery error", "message stuck"]
owner: devops-agent
objective: "Resolve message delivery failures on the Wickr network"
context: "Message delivery depends on sender/recipient connectivity, network health, and client status. AWS cannot access message content due to E2E encryption."
---

## Phase 1 — Triage

MUST:
- Verify network is active: `aws wickr get-network --network-id <id>`
- Check CloudWatch delivery metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Wickr --metric-name MessageDeliveryFailures --dimensions Name=NetworkId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Review CloudTrail for service events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

SHOULD:
- Verify both sender and recipient are active users on the network
- Check if the issue affects all users or specific user pairs
- Review network health metrics for degradation

MAY:
- Check AWS Health Dashboard for Wickr service issues
- Verify client application version is current

## Phase 2 — Remediate

MUST:
- Ensure both sender and recipient have active network accounts
- Verify network status is healthy
- Confirm client applications are connected to the network

SHOULD:
- Have affected users restart their Wickr client application
- Check network connectivity between client and Wickr service endpoints
- Verify message expiration timers are not set too short

MAY:
- Collect client-side logs for detailed analysis
- Test message delivery between different user pairs to isolate the issue

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "client restart: GREEN — non-destructive client operation"
```

## Escalation Conditions

- Message delivery failures affect all users on the network
- Delivery failure rate exceeds 10% for more than 30 minutes
- Messages delivered to wrong recipients (security incident)
- Persistent delivery failures between specific users after client restart

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- `aws cloudwatch get-metric-statistics`: LOW — aggregate delivery metrics
- Message delivery metadata: MEDIUM — reveals communication patterns
- Message content: NOT ACCESSIBLE — end-to-end encrypted

## Prohibited Actions

- NEVER attempt to access or decrypt message content
- NEVER suggest AWS Support can read message content
- NEVER delete user accounts to fix delivery issues
- NEVER modify network encryption settings

## Phase 3 — Rollback

1. If network settings were modified, restore original configuration
2. If user accounts were modified, restore original user settings
3. Verify message delivery resumes normally after changes
4. Confirm no messages were lost during the incident (check delivery receipts)
5. Document the delivery failure incident and resolution

## Common Issues

- symptoms: "Messages show 'sending' but never deliver"
  diagnosis: "Recipient client may be offline or disconnected."
  resolution: "Verify recipient is online. Messages will deliver when recipient reconnects."

- symptoms: "All messages failing for a specific user"
  diagnosis: "User account may be suspended or client authentication expired."
  resolution: "Check user account status. Have user re-authenticate in the Wickr client."

## Output Format

```yaml
root_cause: "message_delivery — <specific_cause>"
evidence:
  - type: delivery_metrics
    content: "<metric data>"
severity: HIGH
mitigation:
  immediate: "Restore message delivery"
  long_term: "Implement delivery monitoring and alerting"
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
