---
title: "F2 — Transit Gateway Route Issues"
description: "Diagnose connectivity failures through Transit Gateway"
status: active
severity: HIGH
triggers:
  - "TGW connectivity failure"
  - "Transit Gateway routing"
  - "Cross-VPC via TGW not working"
owner: devops-agent
objective: "Fix Transit Gateway routing and restore cross-VPC/on-premises connectivity"
context: "TGW has its OWN route tables separate from VPC route tables. Both must be configured. TGW attachments must be associated with a TGW route table. Route propagation can auto-add routes."
---

## Phase 1 — Triage

MUST:
- Check TGW state: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id>`
- List TGW attachments: `aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=<tgw-id>`
- Check TGW route table: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <id> --filters Name=type,Values=static,propagated`
- Check VPC route tables have routes pointing to TGW for cross-VPC CIDRs

SHOULD:
- Verify TGW attachment is in 'available' state
- Check if route propagation is enabled for the attachment
- Verify TGW route table association for each attachment

MAY:
- Check if appliance mode is needed (for stateful inspection)
- Verify cross-region TGW peering if applicable

## Common Issues

- symptoms: "VPC attached to TGW but can't reach other VPCs"
  diagnosis: "VPC route table missing route to destination CIDR → TGW, or TGW route table missing route to destination VPC attachment."
  resolution: "Add routes in both VPC route table (CIDR → TGW) and TGW route table (CIDR → VPC attachment)."

- symptoms: "TGW route table empty despite propagation enabled"
  diagnosis: "Attachment not associated with the TGW route table."
  resolution: "Associate the attachment with the TGW route table, then enable propagation."

## Safety Ratings

```
safety_ratings:
  - "describe-transit-gateways: GREEN — read-only TGW state inspection"
  - "describe-transit-gateway-attachments: GREEN — read-only attachment listing"
  - "search-transit-gateway-routes: GREEN — read-only TGW route inspection"
  - "describe-route-tables: GREEN — read-only VPC route table inspection"
  - "create-transit-gateway-route (add TGW route): YELLOW — adds TGW route, recoverable by deleting"
  - "create-route (add VPC route to TGW): YELLOW — adds VPC route, recoverable by deleting"
  - "enable-transit-gateway-route-table-propagation: YELLOW — enables propagation, recoverable by disabling"
  - "delete-transit-gateway-route: RED — removes TGW route, can break cross-VPC connectivity"
```

## Escalation Conditions

- "Fix requires modifying Transit Gateway route tables"
- "Fix requires modifying production VPC route tables"
- "Multiple VPCs affected by TGW routing issues"
- "Fix involves TGW route propagation changes — affects all attached VPCs"

## Data Sensitivity

- HIGH: TGW route tables (expose cross-VPC and hybrid network architecture)
- HIGH: route table entries (expose network architecture and traffic paths)
- HIGH: VPC flow logs (contain source/destination IPs and ports)
- MEDIUM: TGW attachment configurations, VPC CIDR ranges

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete TGW routes without confirming impact on all attached VPCs"

## Phase 3 — Rollback

- If a TGW route was added: delete it with `aws ec2 delete-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr>`
- If a VPC route to TGW was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <cidr>`
- If route propagation was enabled: disable it with `aws ec2 disable-transit-gateway-route-table-propagation --transit-gateway-route-table-id <tgw-rtb-id> --transit-gateway-attachment-id <attachment-id>`
- Document TGW route tables, VPC route tables, and propagation settings before changes

## Output Format

```yaml
root_cause: "tgw_routes — <detail>"
evidence:
  - type: transit_gateway
    content: "<TGW route table and VPC route table entries>"
severity: HIGH
mitigation:
  immediate: "Add missing routes in TGW and/or VPC route tables"
  long_term: "Use route propagation, document TGW routing design"
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
