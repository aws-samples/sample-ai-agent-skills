---
title: "B2 — Subnet Routing Issues"
description: "Diagnose subnet routing problems with Transit Gateway"
status: active
severity: HIGH
triggers:
  - "subnet cannot reach TGW"
  - "routing not working through TGW"
  - "subnet route table missing TGW"
  - "asymmetric routing"
owner: devops-agent
objective: "Identify and resolve subnet-level routing issues with TGW"
context: "Subnet routing involves both VPC route tables (directing traffic to TGW) and TGW route tables (directing traffic to destination). Both layers must be correctly configured. Different subnets can use different VPC route tables."
---

## Phase 1 — Triage

MUST:
- Identify the subnet's route table: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if the route table has a TGW route: look for routes with TransitGatewayId target
- Verify the TGW attachment includes a subnet in this AZ: `aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <attach-id> --query 'TransitGatewayVpcAttachments[*].SubnetIds'`
- Check TGW route table for return path: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=active`

SHOULD:
- Verify NACLs on the subnet allow the traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Check if the subnet is using the main route table or a custom one
- Verify both source and destination subnets have correct routing

MAY:
- Use VPC Reachability Analyzer: `aws ec2 create-network-insights-path --source <eni-id> --destination <eni-id> --protocol TCP`
- Check VPC Flow Logs for dropped traffic

## Phase 2 — Remediate

MUST:
- Add TGW route to the subnet's route table: `aws ec2 create-route --route-table-id <rtb-id> --destination-cidr-block <cidr> --transit-gateway-id <tgw-id>`
- Ensure TGW route table has return route for the subnet's VPC CIDR
- Fix NACL rules if they block the traffic

SHOULD:
- Use specific CIDR routes rather than 0.0.0.0/0 to TGW when possible
- Ensure consistent routing across all subnets in the VPC

MAY:
- Create dedicated route tables for TGW-attached subnets
- Document the routing architecture

## Common Issues

- symptoms: "Subnet uses main route table which lacks TGW route"
  diagnosis: "Subnet not explicitly associated with a route table that has TGW routes."
  resolution: "Associate subnet with a route table containing TGW routes, or add TGW route to main table."

- symptoms: "Traffic reaches TGW but return path fails"
  diagnosis: "TGW route table missing return route for the source VPC CIDR."
  resolution: "Add route or enable propagation for the source VPC attachment in the TGW route table."

- symptoms: "More specific route overrides TGW route"
  diagnosis: "A more specific CIDR in the VPC route table matches before the TGW route."
  resolution: "Review route table for conflicting more-specific routes. Longest prefix match wins."

## Output Format

```yaml
root_cause: "subnet_routing_issue — <specific_cause>"
evidence:
  - type: subnet_route_table
    content: "<subnet route table entries>"
  - type: tgw_route_table
    content: "<TGW route table entries>"
  - type: nacl_rules
    content: "<NACL configuration>"
severity: HIGH
mitigation:
  immediate: "Fix routing in both VPC and TGW route tables"
  long_term: "Standardize subnet routing with IaC templates"
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
