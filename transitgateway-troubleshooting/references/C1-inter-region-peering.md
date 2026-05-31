---
title: "C1 — Inter-Region Peering Issues"
description: "Diagnose Transit Gateway inter-region peering problems"
status: active
severity: HIGH
triggers:
  - "peering not working"
  - "inter-region connectivity failed"
  - "peering attachment pending"
  - "cross-region TGW"
owner: devops-agent
objective: "Identify and resolve inter-region TGW peering issues"
context: "Inter-region peering connects TGWs in different regions. It does NOT support route propagation — only static routes. Peering requires acceptance by the remote TGW owner. Bandwidth limits apply per region pair."
---

## Phase 1 — Triage

MUST:
- Check peering attachment status: `aws ec2 describe-transit-gateway-peering-attachments --transit-gateway-attachment-ids <attach-id>`
- Verify peering attachment state is available (not pendingAcceptance or failed)
- Check static routes exist in both TGW route tables pointing to the peering attachment: `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=active`
- Verify the peering attachment is associated with the correct route table: `aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <tgw-rtb-id>`

SHOULD:
- Check both sides of the peering (local and remote region)
- Verify no blackhole routes exist for the remote CIDRs
- Confirm the remote TGW is in available state

MAY:
- Check CloudWatch metrics for peering attachment traffic
- Review CloudTrail for peering-related events

## Phase 2 — Remediate

MUST:
- Accept pending peering attachment in the remote region: `aws ec2 accept-transit-gateway-peering-attachment --transit-gateway-attachment-id <attach-id> --region <remote-region>`
- Create static routes in BOTH TGW route tables pointing to the peering attachment:
  `aws ec2 create-transit-gateway-route --transit-gateway-route-table-id <local-rtb-id> --destination-cidr-block <remote-cidr> --transit-gateway-attachment-id <peering-attach-id>`
- Associate the peering attachment with the correct route table

SHOULD:
- Create routes for all CIDRs that need to be reachable across regions
- Verify VPC route tables in both regions also have correct entries

MAY:
- Monitor bandwidth usage on the peering attachment
- Document the inter-region routing architecture

## Common Issues

- symptoms: "Peering attachment stuck in pendingAcceptance"
  diagnosis: "Remote TGW owner has not accepted the peering request."
  resolution: "Accept in remote region: aws ec2 accept-transit-gateway-peering-attachment."

- symptoms: "Peering is available but traffic does not flow"
  diagnosis: "Missing static routes — route propagation is NOT supported for peering."
  resolution: "Create static routes in both TGW route tables pointing to the peering attachment."

- symptoms: "Peering creation failed"
  diagnosis: "TGWs in same region, or ASN conflict, or quota exceeded."
  resolution: "Peering is inter-region only. Ensure unique ASNs. Check peering quota."

## Output Format

```yaml
root_cause: "inter_region_peering_issue — <specific_cause>"
evidence:
  - type: peering_state
    content: "<peering attachment status>"
  - type: static_routes
    content: "<static routes in both TGW route tables>"
  - type: route_table_association
    content: "<peering attachment association>"
severity: HIGH
mitigation:
  immediate: "Accept peering and create static routes"
  long_term: "Automate inter-region routing with IaC"
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
