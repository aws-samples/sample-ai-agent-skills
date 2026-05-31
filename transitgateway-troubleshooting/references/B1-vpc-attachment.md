---
title: "B1 — VPC Attachment Issues"
description: "Diagnose Transit Gateway VPC attachment problems"
status: active
severity: HIGH
triggers:
  - "VPC attachment not working"
  - "VPC cannot reach TGW"
  - "VPC attachment failed"
  - "VPC connectivity through TGW"
owner: devops-agent
objective: "Identify and resolve VPC attachment configuration issues"
context: "VPC attachments connect a VPC to the TGW. Issues include wrong subnets, missing AZs, DNS resolution failures, and appliance mode misconfiguration. Each VPC can have one attachment per TGW."
---

## Phase 1 — Triage

MUST:
- Check VPC attachment details: `aws ec2 describe-transit-gateway-vpc-attachments --filters Name=transit-gateway-id,Values=<tgw-id> Name=vpc-id,Values=<vpc-id>`
- Verify attachment state is available: check State field in output
- List subnets in the attachment: check SubnetIds in attachment details
- Verify VPC route table has route to TGW: `aws ec2 describe-route-tables --filters Name=vpc-id,Values=<vpc-id> --query 'RouteTables[*].Routes[?TransitGatewayId]'`

SHOULD:
- Check which AZs have subnets in the attachment vs which AZs have instances
- Verify DNS support is enabled on the TGW: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].Options.DnsSupport'`
- Check if appliance mode is needed: `aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <attach-id> --query 'TransitGatewayVpcAttachments[*].Options'`

MAY:
- Check security groups and NACLs on instances for TGW traffic
- Verify the TGW attachment subnet has sufficient IP addresses

## Phase 2 — Remediate

MUST:
- Add subnets for missing AZs: `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <attach-id> --add-subnet-ids <subnet-ids>`
- Add VPC route table entries pointing to TGW for destination CIDRs
- Ensure TGW route table has routes for the VPC CIDR

SHOULD:
- Enable appliance mode if routing through a firewall: `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <attach-id> --options ApplianceModeSupport=enable`
- Use dedicated subnets for TGW attachments (not shared with workloads)

MAY:
- Enable DNS support on the TGW if cross-VPC DNS resolution is needed
- Tag the attachment for identification

## Common Issues

- symptoms: "Instances in some AZs cannot reach other VPCs via TGW"
  diagnosis: "VPC attachment missing subnet in the affected AZ."
  resolution: "Add subnet in the missing AZ to the VPC attachment."

- symptoms: "VPC route table has TGW route but traffic still fails"
  diagnosis: "TGW route table missing route for the destination or attachment not associated."
  resolution: "Check TGW route table for destination CIDR and attachment association."

- symptoms: "Stateful firewall dropping return traffic"
  diagnosis: "Appliance mode not enabled on the firewall VPC attachment."
  resolution: "Enable appliance mode: --options ApplianceModeSupport=enable."

## Output Format

```yaml
root_cause: "vpc_attachment_issue — <specific_cause>"
evidence:
  - type: attachment_details
    content: "<VPC attachment state and subnets>"
  - type: vpc_routes
    content: "<VPC route table entries>"
  - type: tgw_routes
    content: "<TGW route table entries>"
severity: HIGH
mitigation:
  immediate: "Fix VPC attachment configuration"
  long_term: "Ensure all AZs have subnets in TGW attachments via IaC"
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
