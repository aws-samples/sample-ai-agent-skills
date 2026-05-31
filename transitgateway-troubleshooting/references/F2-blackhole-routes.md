---
title: "F2 — Blackhole Routes"
description: "Diagnose and resolve blackhole routes in Transit Gateway route tables"
status: active
severity: CRITICAL
triggers:
  - "blackhole route"
  - "traffic being dropped"
  - "route state blackhole"
  - "silent traffic loss"
owner: devops-agent
objective: "Identify and remove blackhole routes that silently drop traffic"
context: "Blackhole routes drop traffic without any error notification. They appear when attachments are deleted but routes remain, or when manually created for traffic isolation. Blackhole routes are a common cause of mysterious connectivity failures."
---

## Phase 1 — Triage

MUST:
- Search for blackhole routes in all TGW route tables:
  `aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <tgw-rtb-id> --filters Name=state,Values=blackhole`
- List all TGW route tables to check each one: `aws ec2 describe-transit-gateway-route-tables --filters Name=transit-gateway-id,Values=<tgw-id>`
- Check if the blackhole route's original attachment still exists: `aws ec2 describe-transit-gateway-attachments --transit-gateway-attachment-ids <attach-id-from-route>`
- Verify the destination CIDR of the blackhole route matches the affected traffic

SHOULD:
- Determine if the blackhole was intentional (traffic isolation) or accidental (deleted attachment)
- Check CloudTrail for attachment deletion events
- Identify if a replacement attachment exists for the same CIDR

MAY:
- Check VPC Flow Logs for dropped traffic matching the blackhole CIDR
- Review all route tables for consistency

## Phase 2 — Remediate

MUST:
- Delete accidental blackhole routes: `aws ec2 delete-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr>`
- Create replacement routes if a new attachment exists for the CIDR:
  `aws ec2 create-transit-gateway-route --transit-gateway-route-table-id <tgw-rtb-id> --destination-cidr-block <cidr> --transit-gateway-attachment-id <new-attach-id>`
- Verify traffic flows after removing the blackhole route

SHOULD:
- Check all route tables for similar blackhole routes
- Enable propagation from the replacement attachment if applicable

MAY:
- Set up CloudWatch alarms for blackhole route detection
- Create automation to clean up blackhole routes after attachment deletion

## Common Issues

- symptoms: "Traffic to a specific CIDR silently fails"
  diagnosis: "Blackhole route exists for that CIDR in the TGW route table."
  resolution: "Delete the blackhole route and create a new route to a valid attachment."

- symptoms: "Blackhole routes appeared after VPC detachment"
  diagnosis: "Propagated routes become blackhole when the source attachment is deleted."
  resolution: "Delete the blackhole routes. Re-enable propagation from the new attachment."

- symptoms: "Intentional blackhole route blocking wanted traffic"
  diagnosis: "Blackhole route was created for isolation but now blocks legitimate traffic."
  resolution: "Delete the blackhole route or create a more specific route to allow the traffic."

## Output Format

```yaml
root_cause: "blackhole_route — <specific_cause>"
evidence:
  - type: blackhole_routes
    content: "<list of blackhole routes found>"
  - type: affected_cidrs
    content: "<CIDRs being dropped>"
  - type: original_attachment
    content: "<status of the original attachment>"
severity: CRITICAL
mitigation:
  immediate: "Delete blackhole routes and create replacement routes"
  long_term: "Automate blackhole route cleanup after attachment deletion"
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
