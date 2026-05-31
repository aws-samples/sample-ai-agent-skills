---
title: "G1 — Multicast Domain Issues"
description: "Diagnose Transit Gateway multicast domain configuration problems"
status: active
severity: MEDIUM
triggers:
  - "multicast not working"
  - "multicast domain creation failed"
  - "multicast domain configuration"
  - "IGMP not working on TGW"
owner: devops-agent
objective: "Identify and resolve TGW multicast domain configuration issues"
context: "TGW multicast requires multicast support enabled at TGW creation time. Multicast domains define the scope of multicast traffic. Domains can use IGMP for dynamic membership or static membership. Not all instance types support multicast."
---

## Phase 1 — Triage

MUST:
- Verify TGW has multicast support enabled: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.MulticastSupport'`
- List multicast domains: `aws ec2 describe-transit-gateway-multicast-domains --filters Name=transit-gateway-id,Values=<tgw-id>`
- Check multicast domain state: `aws ec2 describe-transit-gateway-multicast-domains --transit-gateway-multicast-domain-ids <domain-id>`
- Verify VPC attachments are associated with the multicast domain: `aws ec2 get-transit-gateway-multicast-domain-associations --transit-gateway-multicast-domain-id <domain-id>`

SHOULD:
- Check if IGMP support is configured on the domain
- Verify instance types support multicast (Nitro-based instances)
- Check security groups allow IGMP traffic (protocol 2)

MAY:
- Check multicast group membership: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <domain-id>`
- Review CloudTrail for multicast domain events

## Phase 2 — Remediate

MUST:
- If TGW lacks multicast support, create a new TGW with multicast enabled:
  `aws ec2 create-transit-gateway --options MulticastSupport=enable`
- Create multicast domain:
  `aws ec2 create-transit-gateway-multicast-domain --transit-gateway-id <tgw-id> --options Igmpv2Support=enable,StaticSourcesSupport=enable`
- Associate VPC attachments with the multicast domain

SHOULD:
- Use Nitro-based instance types for multicast
- Allow IGMP (protocol 2) in security groups
- Configure both static and IGMP membership as needed

MAY:
- Set up multicast monitoring
- Document multicast domain architecture

## Common Issues

- symptoms: "Cannot create multicast domain — MulticastNotSupported"
  diagnosis: "TGW was created without multicast support."
  resolution: "Create a new TGW with MulticastSupport=enable. Cannot be enabled on existing TGW."

- symptoms: "Multicast domain created but no traffic flowing"
  diagnosis: "VPC attachments not associated with the domain, or no group members."
  resolution: "Associate attachments and add group members (static or IGMP)."

- symptoms: "IGMP membership not working"
  diagnosis: "IGMP not enabled on domain, or security groups blocking IGMP protocol."
  resolution: "Enable Igmpv2Support on domain. Allow protocol 2 in security groups."

## Output Format

```yaml
root_cause: "multicast_domain_issue — <specific_cause>"
evidence:
  - type: tgw_multicast_support
    content: "<multicast support status on TGW>"
  - type: domain_state
    content: "<multicast domain state>"
  - type: associations
    content: "<VPC attachment associations>"
severity: MEDIUM
mitigation:
  immediate: "Fix multicast domain configuration"
  long_term: "Document multicast requirements and automate with IaC"
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
