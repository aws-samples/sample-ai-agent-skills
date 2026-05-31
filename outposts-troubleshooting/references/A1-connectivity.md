---
title: "A1 — Outpost Rack Connectivity Issues"
description: "Diagnose connectivity problems between the Outpost rack and the parent AWS Region"
status: active
severity: CRITICAL
triggers:
  - "outpost disconnected"
  - "service link down"
  - "outpost not connected"
  - "connectivity lost"
owner: devops-agent
objective: "Restore connectivity between the Outpost rack and the parent AWS Region"
context: "The Outpost connects to the parent Region via a service link. Connectivity issues prevent new instance launches, API calls, and management operations. Running instances continue operating during outages."
---

## Phase 1 — Triage

MUST:
- Check Outpost connectivity status: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Get Outpost details: `aws outposts get-outpost --outpost-id <outpost-id>`
- List all Outposts and their status: `aws outposts list-outposts`
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 10`

SHOULD:
- Verify physical network connectivity to the Outpost rack (customer responsibility)
- Check if the service link VPN tunnels are up
- Verify DNS resolution from the Outpost network
- Check for recent network changes in the on-premises environment

MAY:
- Review network device logs (routers, switches) connecting to the Outpost
- Check for ISP or WAN link issues between the site and AWS Region

## Phase 2 — Remediate

MUST:
- If physical network issue: restore power and network connectivity to the rack
- If service link VPN down: verify the service link configuration and on-premises firewall rules
- Ensure required ports and protocols are allowed through on-premises firewalls (UDP 443 for service link)

SHOULD:
- Verify BGP peering status if using dynamic routing for the service link
- Check for MTU issues on the network path (service link requires 1500 byte MTU minimum)
- Contact AWS Support if service link remains down after verifying physical connectivity

MAY:
- Set up CloudWatch alarms for ConnectedStatus metric to detect future outages early
- Implement redundant network paths to the Outpost rack

## Common Issues

- symptoms: "Outpost shows as disconnected in console"
  diagnosis: "Physical network link between Outpost and upstream network is down."
  resolution: "Verify physical cabling, switch port status, and power to the rack."

- symptoms: "Cannot launch new instances but existing ones are running"
  diagnosis: "Service link is down — existing workloads continue but new operations fail."
  resolution: "Check service link connectivity. Verify firewall rules allow UDP 443 outbound."

## Output Format

```yaml
root_cause: "connectivity — <specific_cause>"
evidence:
  - type: connected_status
    content: "<CloudWatch metric value>"
  - type: outpost_details
    content: "<Outpost configuration>"
severity: CRITICAL
mitigation:
  immediate: "Restore physical or logical connectivity to the Outpost"
  long_term: "Implement redundant network paths and connectivity monitoring"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — connectivity issues prevent access rather than exposing data
- Service Disruption Risk: HIGH — connectivity loss affects all Outpost workloads
- Compliance Impact: HIGH — connectivity is critical for Outpost operations and management

## Escalation Conditions
- Complete loss of connectivity between Outpost and AWS Region
- Service link goes down affecting all Outpost operations
- Local gateway connectivity fails preventing local network access
- Connectivity degradation affects multiple Outpost workloads
- Network equipment failure at the Outpost site

## Data Sensitivity
- Network topology and configurations — INTERNAL
- Service link connection details — INTERNAL
- Local gateway configurations — INTERNAL
- BGP routing information — INTERNAL
- Never expose Outpost site network architecture externally

## Prohibited Actions
- Do NOT modify service link configurations without AWS Support guidance
- Do NOT change local gateway routing during active workloads without approval
- Do NOT bypass network security controls for troubleshooting
- Do NOT expose Outpost management interfaces to untrusted networks
- Do NOT ignore connectivity degradation warnings
- Do NOT modify BGP configurations without network team coordination

## Phase 3 — Rollback
- Restore previous network configurations
- Revert local gateway routing changes
- Re-establish service link connectivity
- Restore BGP peering configurations
- Verify all Outpost workloads are accessible after recovery
- Document connectivity issues and network changes made

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
