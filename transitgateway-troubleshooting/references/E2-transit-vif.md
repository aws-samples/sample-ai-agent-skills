---
title: "E2 — Transit VIF Issues"
description: "Diagnose Transit Virtual Interface configuration problems"
status: active
severity: HIGH
triggers:
  - "transit VIF not working"
  - "transit virtual interface"
  - "DX VIF for TGW"
  - "transit VIF down"
owner: devops-agent
objective: "Identify and resolve transit VIF configuration issues for TGW connectivity"
context: "Transit VIFs are required to connect a Direct Connect connection to a DX gateway that is associated with a TGW. Transit VIFs support 802.1Q VLAN tagging and BGP. They differ from private VIFs (which connect to VGWs) and public VIFs."
---

## Phase 1 — Triage

MUST:
- List virtual interfaces on the DX connection: `aws directconnect describe-virtual-interfaces --connection-id <dx-conn-id>`
- Check transit VIF state: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[*].{Id:virtualInterfaceId,State:virtualInterfaceState,Type:virtualInterfaceType,DxGw:directConnectGatewayId}'`
- Verify the transit VIF is attached to the correct DX gateway
- Check BGP peer status on the transit VIF: `aws directconnect describe-virtual-interfaces --virtual-interface-id <vif-id> --query 'virtualInterfaces[*].bgpPeers'`

SHOULD:
- Verify VLAN ID matches on both AWS and on-premises router
- Check BGP ASN configuration matches
- Verify jumbo frame support if using MTU 8500

MAY:
- Check DX connection status: `aws directconnect describe-connections --connection-id <dx-conn-id>`
- Review CloudTrail for VIF creation/modification events

## Phase 2 — Remediate

MUST:
- Create transit VIF if missing:
  `aws directconnect create-transit-virtual-interface --connection-id <dx-conn-id> --new-transit-virtual-interface virtualInterfaceName=<name>,vlan=<vlan>,asn=<customer-asn>,directConnectGatewayId=<dx-gw-id>`
- Ensure BGP peer is configured and up
- Verify VLAN tagging matches on both sides

SHOULD:
- Enable jumbo frames (MTU 8500) if supported by the DX connection
- Configure BGP authentication if required
- Set up redundant transit VIFs on separate DX connections

MAY:
- Enable BFD for faster failover detection
- Monitor VIF metrics in CloudWatch

## Common Issues

- symptoms: "Transit VIF in down state"
  diagnosis: "BGP session not established or VLAN mismatch."
  resolution: "Verify VLAN ID and BGP configuration on both AWS and on-premises sides."

- symptoms: "Created private VIF instead of transit VIF"
  diagnosis: "Private VIFs connect to VGWs, not DX gateways for TGW."
  resolution: "Delete the private VIF and create a transit VIF attached to the DX gateway."

- symptoms: "BGP peer stuck in pending state"
  diagnosis: "On-premises router not configured for BGP peering."
  resolution: "Configure BGP on the on-premises router with the correct peer IP and ASN."

## Output Format

```yaml
root_cause: "transit_vif_issue — <specific_cause>"
evidence:
  - type: vif_status
    content: "<transit VIF state and BGP peers>"
  - type: dx_gateway
    content: "<DX gateway association>"
  - type: bgp_config
    content: "<BGP configuration details>"
severity: HIGH
mitigation:
  immediate: "Fix transit VIF or BGP configuration"
  long_term: "Set up redundant transit VIFs with monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Route tables reveal network topology across accounts
- Peering configurations expose cross-region connectivity
- VPN attachment details contain tunnel configurations
- Classification: INTERNAL - network architecture and routing topology

## Prohibited Actions

- NEVER delete transit gateway route tables with active associations
- NEVER remove VPC or VPN attachments without confirming traffic rerouting
- NEVER modify route propagation settings during business hours without approval
- NEVER change peering configurations without coordinating with peer account owners

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest removing route table associations without review"
  - "NEVER suggest disabling route propagation"
  - "NEVER modify production TGW without maintenance window"
