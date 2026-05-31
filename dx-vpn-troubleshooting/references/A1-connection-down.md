---
title: "A1 â€” Direct Connect Connection Down"
description: "Diagnose a Direct Connect connection in down or degraded state"
status: active
severity: CRITICAL
triggers:
  - "DX connection down"
  - "Direct Connect down"
  - "connection state down"
  - "ConnectionState 0"
owner: devops-agent
objective: "Identify why the DX connection is down and restore connectivity"
context: "A DX connection can be down due to physical layer issues (fiber cut, port failure), LOA-CFA not completed, cross-connect not provisioned, or AWS-side maintenance. The ConnectionState CloudWatch metric reports 1 for up and 0 for down."
---

## Phase 1 â€” Triage

MUST:
- Check connection state: `aws directconnect describe-connections --connection-id <dxcon-id> --query 'connections[].connectionState'`
- Check connection details: `aws directconnect describe-connections --connection-id <dxcon-id>`
- Check CloudWatch ConnectionState metric: `aws cloudwatch get-metric-statistics --namespace AWS/DX --metric-name ConnectionState --dimensions Name=ConnectionId,Value=<dxcon-id> --start-time <time> --end-time <time> --period 300 --statistics Average`
- Verify the connection is not in "ordering" or "requested" state (not yet provisioned)

SHOULD:
- Check if the connection is part of a LAG: `aws directconnect describe-lags`
- Check AWS Health Dashboard for DX service events in the connection's region
- Verify the cross-connect is physically completed at the colocation facility

MAY:
- Check CloudTrail for recent DX API calls (DeleteConnection, etc.)
- Contact the colocation provider to verify physical layer status

## Phase 2 â€” Remediate

MUST:
- If state is "ordering": complete the LOA-CFA process with the colocation provider
- If state is "down" after being "available": check physical layer (fiber, port) with colocation provider
- If state is "deleted": the connection was removed â€” create a new connection

SHOULD:
- Verify light levels on both ends of the cross-connect
- Check for interface errors on the customer router (CRC errors, input errors)

MAY:
- Request an AWS DX connection test from the console
- Open an AWS Support case for persistent down state

## Common Issues

- symptoms: "Connection state is 'ordering' for days"
  diagnosis: "LOA-CFA not completed. Cross-connect not provisioned at the colocation facility."
  resolution: "Download the LOA-CFA from the console and provide it to the colocation provider to complete the cross-connect."

- symptoms: "Connection was 'available' but now shows 'down'"
  diagnosis: "Physical layer failure â€” fiber cut, port failure, or colocation issue."
  resolution: "Check physical layer with colocation provider. Verify light levels. Check for interface errors on customer router."

- symptoms: "Connection shows 'available' but no traffic flows"
  diagnosis: "Layer 1 is up but no VIF is configured, or BGP is not established."
  resolution: "Create a VIF on the connection and configure BGP. See A2 and A3 runbooks."

## Output Format

```yaml
root_cause: "connection_down â€” <specific_cause>"
evidence:
  - type: connection_state
    content: "<connection state and details>"
  - type: cloudwatch_metrics
    content: "<ConnectionState metric values>"
severity: CRITICAL
mitigation:
  immediate: "Restore physical connectivity or complete provisioning"
  long_term: "Implement redundant DX connections or DX+VPN failover"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Direct Connect and VPN provide critical network connectivity |
| Data Risk | HIGH — Network path changes can disrupt all traffic flows |
| Recovery Complexity | HIGH — Physical cross-connects and BGP sessions require coordination |
| Blast Radius | CRITICAL — Affects all workloads dependent on hybrid connectivity |

## Escalation Conditions

- Connection down with no redundant path available — escalate immediately to network operations
- BGP session flapping causing route instability across multiple VIFs
- Physical layer issues requiring colocation provider coordination
- MACsec or IPsec negotiation failures blocking encrypted traffic
- Transit Gateway attachment issues affecting multiple VPCs
- Failover not triggering between DX and VPN backup paths

## Data Sensitivity

- BGP route advertisements may expose internal network topology
- VPN pre-shared keys and IKE credentials must never be logged or displayed
- Connection and VIF IDs should be treated as sensitive infrastructure identifiers
- MACsec secret keys (CKN/CAK) must be handled with encryption-at-rest standards
- Customer router configurations may contain proprietary network architecture details

## PROHIBITED Actions

- NEVER suggest deleting a Direct Connect connection without confirming backup connectivity
- NEVER modify BGP route advertisements without understanding the full routing topology
- NEVER change VPN tunnel configurations on both tunnels simultaneously
- NEVER disable MACsec encryption on a connection that requires encrypted transit
- NEVER delete a Virtual Interface (VIF) that is actively carrying production traffic
- NEVER modify Transit Gateway route tables without verifying all dependent attachments

## Phase 3 — Rollback

- If BGP changes cause route instability: revert route advertisements to previous AS-path and prefix configuration
- If VPN tunnel changes break connectivity: restore previous IKE/IPsec parameters from backup configuration
- If VIF deletion was premature: recreate VIF with identical VLAN, BGP ASN, and auth key settings
- If failover configuration changes fail: restore previous DX/VPN failover priority and health check settings
- If Transit Gateway route changes cause blackholing: revert route table associations and propagations
- Document all rollback actions in the incident timeline with before/after routing state
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling MACsec encryption"
  - "NEVER suggest removing BGP authentication"
  - "NEVER modify production VPN tunnels without maintenance window"
