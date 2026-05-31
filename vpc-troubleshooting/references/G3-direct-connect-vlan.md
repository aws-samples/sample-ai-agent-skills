---
title: "G3 — Direct Connect VLAN Issues"
description: "Diagnose Direct Connect virtual interface and VLAN configuration issues"
status: active
severity: HIGH
triggers:
  - "Direct Connect virtual interface down"
  - "VLAN mismatch"
  - "DX connectivity failure"
owner: devops-agent
objective: "Restore Direct Connect virtual interface connectivity"
context: "Direct Connect uses virtual interfaces (VIFs) over a physical connection. Each VIF has a VLAN ID, peer IPs, and BGP ASN. Private VIF connects to VPC (via VGW/DXGW), public VIF connects to AWS public services, transit VIF connects to TGW."
---

## Phase 1 — Triage

MUST:
- Check DX connection state: `aws directconnect describe-connections`
- Check virtual interface state: `aws directconnect describe-virtual-interfaces`
- Verify VLAN ID matches on both AWS and on-premises sides
- Check BGP peer status on the virtual interface

SHOULD:
- Verify the VIF is associated with the correct VGW, DXGW, or TGW
- Check if the physical connection is up (connectionState=available)
- Verify peer IP addresses and BGP ASN match

MAY:
- Check LOA-CFA was properly implemented by the colocation provider
- Verify jumbo frame support if MTU 9001 is configured
- Check if the VIF was recently created (can take minutes to provision)

## Common Issues

- symptoms: "VIF state is 'down', physical connection is 'available'"
  diagnosis: "VLAN ID mismatch between AWS VIF and on-premises router, or BGP not established."
  resolution: "Verify VLAN ID matches, check BGP configuration, verify peer IPs."

- symptoms: "Private VIF up but can't reach VPC resources"
  diagnosis: "VIF not associated with VGW, or VGW not attached to VPC, or missing route propagation."
  resolution: "Associate VIF with VGW, attach VGW to VPC, enable route propagation."

## Safety Ratings

```
safety_ratings:
  - "describe-connections: GREEN — read-only DX connection state check"
  - "describe-virtual-interfaces: GREEN — read-only VIF state and BGP check"
  - "describe-virtual-gateways: GREEN — read-only VGW state check"
  - "create-private-virtual-interface: YELLOW — creates new VIF, recoverable by deleting"
  - "associate-virtual-interface (associate VIF with VGW/DXGW): YELLOW — creates association, recoverable by disassociating"
  - "delete-virtual-interface: RED — deletes VIF, breaks DX connectivity for that VIF"
```

## Escalation Conditions

- "Fix requires coordination with colocation provider or on-premises network team"
- "Physical DX connection is down — requires provider intervention"
- "Fix requires creating or deleting virtual interfaces in production"
- "Fix involves VLAN ID changes — requires matching changes on both sides"

## Data Sensitivity

- HIGH: DX connection details (expose physical connectivity and colocation information)
- HIGH: VIF configuration (expose VLAN IDs, peer IPs, BGP ASN)
- MEDIUM: VGW/DXGW associations, MTU settings, LOA-CFA details

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a DX virtual interface without confirming backup connectivity exists"

## Phase 3 — Rollback

- If a new VIF was created: delete it with `aws directconnect delete-virtual-interface --virtual-interface-id <vif-id>`
- If a VIF was associated with a VGW: the association is part of VIF creation — delete and recreate VIF to change
- If route propagation was enabled on VGW: disable it with `aws ec2 disable-vgw-route-propagation --gateway-id <vgw-id> --route-table-id <rtb-id>`
- If MTU was changed: revert on-premises device to original MTU setting
- Document DX connection state, VIF configuration, VLAN IDs, and BGP settings before changes

## Output Format

```yaml
root_cause: "direct_connect_vlan — <detail>"
evidence:
  - type: direct_connect
    content: "<connection state, VIF state, VLAN configuration>"
severity: HIGH
mitigation:
  immediate: "Fix VLAN/BGP configuration mismatch"
  long_term: "Use redundant DX connections, monitor with CloudWatch"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
