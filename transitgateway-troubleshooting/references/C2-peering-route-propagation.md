---
title: "C2 — Peering Route Propagation Issues"
description: "Diagnose route propagation issues with TGW peering attachments"
status: active
severity: HIGH
triggers:
  - "peering routes not propagating"
  - "remote CIDRs not in route table"
  - "peering route missing"
  - "propagation not working for peering"
owner: devops-agent
objective: "Clarify that peering does not support propagation and implement static routes"
context: "Inter-region TGW peering does NOT support automatic route propagation. This is a common misconception. All routes for peered TGWs must be static. This runbook helps users understand and implement the correct approach."
---

## Phase 1 — Triage

MUST:
- Confirm the attachment type is peering: `aws ec2 describe-transit-gateway-attachments --transit-gateway-attachment-ids <attach-id> --query 'TransitGatewayAttachments[*].ResourceType'`
- Check if propagation was attempted on the peering attachment: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`
- List existing static routes for the peering attachment: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=type,Values=static`
- Identify all CIDRs that need to be reachable across the peering

SHOULD:
- Check the remote TGW's route table for return static routes
- Verify the peering attachment is in available state
- Confirm the peering attachment is associated with the correct route table

MAY:
- List all VPC CIDRs attached to the remote TGW to determine needed static routes
- Check if a summary route can cover multiple remote CIDRs

## Phase 2 — Remediate

MUST:
- Create static routes for each remote CIDR:
  `aws ec2 create-transit-gateway-route --transit-gateway-route-table-id <local-rtb-id> --destination-cidr-block <remote-cidr> --transit-gateway-attachment-id <peering-attach-id>`
- Create corresponding return routes in the remote TGW route table
- Do NOT attempt to enable propagation on peering attachments

SHOULD:
- Use summary CIDRs where possible to reduce the number of static routes
- Document all static routes for peering in IaC

MAY:
- Set up monitoring for route table changes
- Create automation to sync routes when new VPCs are added

## Common Issues

- symptoms: "EnableTransitGatewayRouteTablePropagation fails for peering attachment"
  diagnosis: "Route propagation is not supported for inter-region peering attachments."
  resolution: "Use static routes instead. Create a route for each remote CIDR."

- symptoms: "Some remote CIDRs reachable but not others"
  diagnosis: "Static routes only created for some CIDRs."
  resolution: "Add static routes for all remote CIDRs that need to be reachable."

- symptoms: "Traffic works one direction but not the other"
  diagnosis: "Static routes only created in one TGW route table."
  resolution: "Create static routes in BOTH the local and remote TGW route tables."

## Output Format

```yaml
root_cause: "peering_route_propagation — propagation_not_supported"
evidence:
  - type: attachment_type
    content: "peering — propagation not supported"
  - type: existing_static_routes
    content: "<current static routes>"
  - type: missing_routes
    content: "<CIDRs without static routes>"
severity: HIGH
mitigation:
  immediate: "Create static routes for all required CIDRs"
  long_term: "Automate static route management for peering with IaC"
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
