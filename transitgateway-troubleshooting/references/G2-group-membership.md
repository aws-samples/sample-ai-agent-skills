---
title: "G2 — Multicast Group Membership Issues"
description: "Diagnose Transit Gateway multicast group membership problems"
status: active
severity: MEDIUM
triggers:
  - "multicast group not receiving"
  - "group membership failed"
  - "multicast member not working"
  - "static group member"
owner: devops-agent
objective: "Identify and resolve multicast group membership issues"
context: "Multicast group membership determines which ENIs receive multicast traffic. Members can be added statically or via IGMP. Sources send multicast traffic to group addresses. Both sources and members must be registered in the multicast domain."
---

## Phase 1 — Triage

MUST:
- Search for group members: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <domain-id>`
- Check if the ENI is registered as a member: look for the specific ENI in group results
- Verify the multicast group address is correct (224.0.0.0/4 range)
- Check if the source ENI is registered: `aws ec2 search-transit-gateway-multicast-groups --transit-gateway-multicast-domain-id <domain-id> --filters Name=is-group-source,Values=true`

SHOULD:
- Verify the ENI's instance type supports multicast
- Check if IGMP membership is being sent by the instance
- Verify security groups allow multicast traffic on the required ports

MAY:
- Check if the multicast domain has the correct IGMP settings
- Review instance-level multicast configuration

## Phase 2 — Remediate

MUST:
- Register static group members:
  `aws ec2 register-transit-gateway-multicast-group-members --transit-gateway-multicast-domain-id <domain-id> --group-ip-address <multicast-ip> --network-interface-ids <eni-ids>`
- Register multicast sources:
  `aws ec2 register-transit-gateway-multicast-group-sources --transit-gateway-multicast-domain-id <domain-id> --group-ip-address <multicast-ip> --network-interface-ids <source-eni-ids>`
- Ensure the ENI's subnet is in a VPC attachment associated with the multicast domain

SHOULD:
- Configure IGMP on instances for dynamic membership
- Allow multicast traffic in security groups (destination IP in 224.0.0.0/4)

MAY:
- Set up monitoring for multicast group membership changes
- Document multicast group architecture

## Common Issues

- symptoms: "Instance not receiving multicast traffic"
  diagnosis: "ENI not registered as a group member in the multicast domain."
  resolution: "Register the ENI as a group member: register-transit-gateway-multicast-group-members."

- symptoms: "Multicast source registered but members not receiving"
  diagnosis: "Members not registered, or security groups blocking multicast."
  resolution: "Register members and allow multicast traffic in security groups."

- symptoms: "IGMP join not being recognized"
  diagnosis: "IGMP support not enabled on the multicast domain."
  resolution: "Verify Igmpv2Support=enable on the domain. Use static membership as fallback."

## Output Format

```yaml
root_cause: "group_membership_issue — <specific_cause>"
evidence:
  - type: group_members
    content: "<registered group members>"
  - type: group_sources
    content: "<registered multicast sources>"
  - type: domain_config
    content: "<multicast domain IGMP settings>"
severity: MEDIUM
mitigation:
  immediate: "Register missing group members or sources"
  long_term: "Automate multicast membership management"
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
