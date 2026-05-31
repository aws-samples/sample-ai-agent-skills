---
title: "B3 — Cross-AZ Connectivity Issues"
description: "Diagnose cross-AZ connectivity problems through Transit Gateway"
status: active
severity: MEDIUM
triggers:
  - "cross-AZ traffic failing"
  - "some AZs cannot reach TGW"
  - "AZ-specific connectivity issue"
  - "cross-AZ latency"
owner: devops-agent
objective: "Identify and resolve cross-AZ connectivity issues through TGW"
context: "TGW VPC attachments need subnets in each AZ for direct connectivity. Missing AZ subnets force traffic to cross AZ boundaries within the VPC, adding latency and cost. Appliance mode affects AZ routing behavior."
---

## Phase 1 — Triage

MUST:
- Check which AZs have subnets in the TGW attachment: `aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <attach-id> --query 'TransitGatewayVpcAttachments[*].SubnetIds'`
- Map subnets to AZs: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone}'`
- Identify which AZ the affected instances are in: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone,Subnet:SubnetId}'`
- Check if appliance mode is enabled: `aws ec2 describe-transit-gateway-vpc-attachments --transit-gateway-attachment-ids <attach-id> --query 'TransitGatewayVpcAttachments[*].Options'`

SHOULD:
- Verify the destination VPC attachment also has subnets in matching AZs
- Check for AZ-specific NACL or security group differences
- Review CloudWatch metrics per AZ for the TGW

MAY:
- Check cross-AZ data transfer costs in Cost Explorer
- Use VPC Flow Logs filtered by AZ to identify traffic patterns

## Phase 2 — Remediate

MUST:
- Add subnets in missing AZs: `aws ec2 modify-transit-gateway-vpc-attachment --transit-gateway-attachment-id <attach-id> --add-subnet-ids <subnet-ids>`
- Ensure both source and destination VPC attachments cover the same AZs

SHOULD:
- Use dedicated /28 subnets for TGW attachments in each AZ
- Enable appliance mode only when routing through stateful appliances

MAY:
- Monitor cross-AZ data transfer costs
- Consider AZ affinity for latency-sensitive workloads

## Common Issues

- symptoms: "Instances in AZ-b cannot reach resources in other VPCs"
  diagnosis: "TGW VPC attachment has no subnet in AZ-b."
  resolution: "Add a subnet in AZ-b to the TGW VPC attachment."

- symptoms: "Intermittent connectivity through firewall VPC"
  diagnosis: "Appliance mode not enabled, causing asymmetric AZ routing."
  resolution: "Enable appliance mode on the firewall VPC attachment."

- symptoms: "Higher latency for some instances"
  diagnosis: "Instances crossing AZ boundaries to reach TGW attachment subnet."
  resolution: "Add TGW attachment subnet in the same AZ as the affected instances."

## Output Format

```yaml
root_cause: "cross_az_issue — <specific_cause>"
evidence:
  - type: attachment_subnets
    content: "<subnets and their AZs in the attachment>"
  - type: instance_placement
    content: "<affected instance AZ placement>"
  - type: appliance_mode
    content: "<appliance mode status>"
severity: MEDIUM
mitigation:
  immediate: "Add subnets in missing AZs"
  long_term: "Ensure all VPC attachments cover all AZs via IaC"
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
