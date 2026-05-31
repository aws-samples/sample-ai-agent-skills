---
title: "F3 — Route Table Association Issues"
description: "Diagnose Transit Gateway route table association problems"
status: active
severity: HIGH
triggers:
  - "wrong route table associated"
  - "association failed"
  - "attachment using wrong routes"
  - "route table association error"
owner: devops-agent
objective: "Identify and fix route table association issues for TGW attachments"
context: "Each TGW attachment must be associated with exactly one route table. The associated route table determines routing for traffic FROM that attachment. Wrong associations cause traffic to be routed incorrectly. Auto-association uses the default route table."
---

## Phase 1 — Triage

MUST:
- Check current association for the attachment: `aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <tgw-rtb-id>`
- List all route tables and their associations: iterate over all TGW route tables
- Verify which route table the attachment SHOULD be associated with
- Check if auto-association is enabled: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.DefaultRouteTableAssociation'`

SHOULD:
- Verify the associated route table has the correct routes for this attachment's traffic
- Check if the attachment was recently created and auto-associated with the default table
- Compare the intended routing design with actual associations

MAY:
- Check CloudTrail for association change events
- Review the network segmentation design

## Phase 2 — Remediate

MUST:
- Disassociate from the wrong route table:
  `aws ec2 disassociate-transit-gateway-route-table --transit-gateway-route-table-id <wrong-rtb-id> --transit-gateway-attachment-id <attach-id>`
- Associate with the correct route table:
  `aws ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id <correct-rtb-id> --transit-gateway-attachment-id <attach-id>`
- Verify traffic flows correctly after the change

SHOULD:
- Disable auto-association if using custom route tables for segmentation
- Update IaC to reflect the correct associations

MAY:
- Document the route table association design
- Set up monitoring for association changes

## Common Issues

- symptoms: "New attachment auto-associated with default route table instead of custom"
  diagnosis: "Auto-association is enabled on the TGW."
  resolution: "Disable auto-association and manually associate with the correct route table."

- symptoms: "Cannot associate attachment — already associated"
  diagnosis: "Attachment is associated with another route table (one association per attachment)."
  resolution: "Disassociate from the current table first, then associate with the new one."

- symptoms: "Traffic from attachment uses wrong routing"
  diagnosis: "Attachment associated with the wrong route table."
  resolution: "Change association to the correct route table."

## Output Format

```yaml
root_cause: "route_table_association_issue — <specific_cause>"
evidence:
  - type: current_association
    content: "<current route table association>"
  - type: expected_association
    content: "<intended route table>"
  - type: auto_association
    content: "<auto-association setting>"
severity: HIGH
mitigation:
  immediate: "Fix route table association"
  long_term: "Disable auto-association and manage via IaC"
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
