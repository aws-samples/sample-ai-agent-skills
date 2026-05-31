---
title: "A2 — TGW Route Table Issues"
description: "Diagnose Transit Gateway route table configuration problems"
status: active
severity: HIGH
triggers:
  - "TGW route table not working"
  - "routes not appearing"
  - "route table misconfigured"
  - "cannot create route table"
owner: devops-agent
objective: "Identify and resolve TGW route table configuration issues"
context: "TGW route tables control traffic flow between attachments. Issues include missing routes, incorrect associations, propagation failures, and confusion with VPC route tables. TGW route tables are separate from VPC route tables."
---

## Phase 1 — Triage

MUST:
- List all TGW route tables: `aws ec2 describe-transit-gateway-route-tables --filters Name=transit-gateway-id,Values=<tgw-id>`
- Search for all routes in the affected route table: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=active,blackhole`
- Check route table associations: `aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <tgw-rtb-id>`
- Check route table propagations: `aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <tgw-rtb-id>`

SHOULD:
- Verify the TGW default route table settings: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.{AutoAssoc:DefaultRouteTableAssociation,AutoProp:DefaultRouteTablePropagation}'`
- Check if the correct route table is the default: compare default route table ID with the one being investigated
- Verify VPC route tables also have entries pointing to the TGW: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id>`

MAY:
- Check CloudTrail for recent route table modifications
- Compare route tables across multiple TGW route tables for consistency

## Phase 2 — Remediate

MUST:
- Add missing static routes: `aws ec2 create-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr> --transit-gateway-attachment-id <attach-id>`
- Fix associations if attachments are associated with the wrong route table
- Enable propagation from required attachments

SHOULD:
- Remove stale or blackhole routes: `aws ec2 delete-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr>`
- Ensure VPC route tables also have correct entries pointing to the TGW

MAY:
- Create separate route tables for network segmentation
- Document the routing architecture for future reference

## Common Issues

- symptoms: "Traffic not flowing between VPCs"
  diagnosis: "TGW route table missing routes or attachments not associated."
  resolution: "Verify both association and propagation/static routes in the TGW route table."

- symptoms: "Routes appear in TGW but traffic still fails"
  diagnosis: "VPC route tables missing entry pointing to TGW."
  resolution: "Add route in VPC route table: destination CIDR → target tgw-*."

- symptoms: "Default route table has all routes but segmentation needed"
  diagnosis: "Auto-association and auto-propagation are enabled on default table."
  resolution: "Disable auto-association/propagation. Create custom route tables for segmentation."

## Output Format

```yaml
root_cause: "route_table_issue — <specific_cause>"
evidence:
  - type: route_table_routes
    content: "<routes found in TGW route table>"
  - type: associations
    content: "<attachment associations>"
  - type: propagations
    content: "<propagation configuration>"
severity: HIGH
mitigation:
  immediate: "Fix route table configuration"
  long_term: "Implement route table management via IaC"
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
