---
title: "F1 — Route Propagation Issues"
description: "Diagnose Transit Gateway route propagation failures"
status: active
severity: HIGH
triggers:
  - "routes not propagating"
  - "propagation not working"
  - "VPC CIDR not in TGW route table"
  - "dynamic routes missing"
owner: devops-agent
objective: "Identify and resolve route propagation issues in TGW route tables"
context: "Route propagation automatically adds routes from attachments to TGW route tables. VPC attachments propagate their CIDR, VPN attachments propagate BGP-learned routes. Propagation must be explicitly enabled per attachment per route table. Inter-region peering does NOT support propagation."
---

## Phase 1 — Triage

MUST:
- Check propagation settings on the route table: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`
- Verify the attachment is in available state: `aws ec2 describe-transit-gateway-attachments --transit-gateway-attachment-ids <attach-id>`
- Search for propagated routes: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=type,Values=propagated`
- Check if a static route overrides the propagated route: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=type,Values=static`

SHOULD:
- Verify the attachment type supports propagation (VPC, VPN, DX — NOT peering)
- Check if auto-propagation is enabled on the default route table
- Verify the VPC CIDR is correct: `aws ec2 describe-vpcs --vpc-ids <vpc-id> --query 'Vpcs[*].CidrBlockAssociationSet'`

MAY:
- Check CloudTrail for propagation enable/disable events
- Compare propagated routes across multiple route tables

## Phase 2 — Remediate

MUST:
- Enable propagation from the attachment to the route table:
  `aws ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id <tgw-rtb-id> --transit-gateway-attachment-id <attach-id>`
- Remove conflicting static routes if they override desired propagated routes
- Ensure the attachment is in available state

SHOULD:
- Enable propagation to all route tables that need the routes
- Verify propagated routes appear after enabling

MAY:
- Set up monitoring for route table changes
- Document propagation configuration in IaC

## Common Issues

- symptoms: "VPC CIDR not appearing in TGW route table"
  diagnosis: "Propagation not enabled from the VPC attachment to this route table."
  resolution: "Enable propagation: aws ec2 enable-transit-gateway-route-table-propagation."

- symptoms: "Propagated route exists but traffic uses wrong path"
  diagnosis: "A static route with the same CIDR overrides the propagated route."
  resolution: "Remove the conflicting static route or update it to point to the correct attachment."

- symptoms: "Propagation enabled but routes still missing"
  diagnosis: "Attachment not in available state, or attachment type doesn't support propagation."
  resolution: "Verify attachment state. Peering attachments do NOT support propagation."

## Output Format

```yaml
root_cause: "route_propagation_issue — <specific_cause>"
evidence:
  - type: propagation_config
    content: "<propagation settings>"
  - type: propagated_routes
    content: "<routes propagated to the table>"
  - type: static_overrides
    content: "<static routes that may conflict>"
severity: HIGH
mitigation:
  immediate: "Enable propagation or fix conflicting routes"
  long_term: "Manage propagation settings via IaC"
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
