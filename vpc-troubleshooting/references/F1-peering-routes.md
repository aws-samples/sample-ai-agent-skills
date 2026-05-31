---
title: "F1 — VPC Peering Route Issues"
description: "Diagnose connectivity failures across VPC peering connections"
status: active
severity: HIGH
triggers:
  - "Can't reach peered VPC"
  - "Peering connection active but no connectivity"
  - "Cross-VPC traffic failing"
owner: devops-agent
objective: "Fix VPC peering routing and restore cross-VPC connectivity"
context: "VPC peering requires routes in BOTH VPCs pointing to the peering connection. Peering is non-transitive. CIDRs must not overlap. SG references work within same region only."
---

## Phase 1 — Triage

MUST:
- Check peering connection state: `aws ec2 describe-vpc-peering-connections --vpc-peering-connection-ids <pcx-id>` → Status
- Verify routes exist in BOTH VPCs' route tables pointing to the peering connection
- Check for CIDR overlap between the two VPCs
- Verify security groups allow traffic from the peer VPC CIDR

SHOULD:
- Check NACLs on both sides allow the traffic
- Verify DNS resolution across peering (requester/accepter DNS resolution options)
- Check if the peering connection allows the specific traffic type

MAY:
- Use Reachability Analyzer across the peering connection
- Check if the peering was recently re-created (routes may reference old pcx-id)

## Common Issues

- symptoms: "Peering active but can't reach instances in peer VPC"
  diagnosis: "Route table in one or both VPCs missing route to peer CIDR → pcx-xxx."
  resolution: "Add routes in both VPCs: peer CIDR → peering connection ID."

- symptoms: "DNS names from peer VPC don't resolve"
  diagnosis: "DNS resolution not enabled on the peering connection."
  resolution: "Modify peering connection to enable DNS resolution for requester/accepter."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-peering-connections: GREEN — read-only peering status check"
  - "describe-route-tables: GREEN — read-only route table inspection"
  - "describe-security-groups: GREEN — read-only SG rule inspection"
  - "create-route (add peering route): YELLOW — adds route to peering connection, recoverable by deleting"
  - "modify-vpc-peering-connection-options (enable DNS resolution): YELLOW — changes peering option, recoverable by disabling"
  - "authorize-security-group-ingress (allow peer CIDR): YELLOW — adds SG rule, recoverable by revoking"
```

## Escalation Conditions

- "Fix requires modifying production VPC route tables"
- "Fix requires adding routes in BOTH VPCs (coordination needed)"
- "Multiple subnets or AZs affected by missing peering routes"
- "Fix involves modifying security groups to allow cross-VPC traffic"

## Data Sensitivity

- HIGH: route table entries (expose network architecture and cross-VPC paths)
- HIGH: security group rules (expose cross-VPC access patterns)
- HIGH: VPC peering configurations (expose inter-VPC connectivity)
- MEDIUM: subnet CIDR ranges, DNS resolution settings

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER delete a VPC peering connection without confirming both sides are aware"

## Phase 3 — Rollback

- If a peering route was added: delete it with `aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <peer-cidr>`
- If DNS resolution was enabled on peering: disable it with `aws ec2 modify-vpc-peering-connection-options --vpc-peering-connection-id <pcx-id> --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=false`
- If a security group rule was added: revoke it with `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol <proto> --port <port> --cidr <peer-cidr>`
- Document route tables and SG rules in BOTH VPCs before making changes

## Output Format

```yaml
root_cause: "peering_routes — <detail>"
evidence:
  - type: route_table
    content: "<route tables from both VPCs>"
severity: HIGH
mitigation:
  immediate: "Add missing routes in both VPCs"
  long_term: "Use IaC to manage peering routes, consider TGW for many-to-many connectivity"
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
