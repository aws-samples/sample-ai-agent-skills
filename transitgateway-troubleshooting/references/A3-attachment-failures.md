---
title: "A3 — TGW Attachment Failures"
description: "Diagnose Transit Gateway attachment creation and state failures"
status: active
severity: HIGH
triggers:
  - "attachment failed"
  - "attachment pending"
  - "cannot attach to TGW"
  - "attachment rejected"
owner: devops-agent
objective: "Identify why TGW attachments are failing and resolve the issue"
context: "TGW attachments connect VPCs, VPNs, Direct Connect gateways, peering connections, and Connect peers to the TGW. Failures can occur due to permissions, quotas, subnet issues, cross-account acceptance, or incompatible configurations."
---

## Phase 1 — Triage

MUST:
- Check attachment status: `aws ec2 describe-transit-gateway-attachments --transit-gateway-attachment-ids <attach-id>`
- List all attachments on the TGW: `aws ec2 describe-transit-gateway-attachments --filters Name=transit-gateway-id,Values=<tgw-id>`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names ec2:CreateTransitGatewayVpcAttachment`
- Check attachment quota: `aws service-quotas get-service-quota --service-code vpc --quota-code L-A1B6A0AB`

SHOULD:
- Check CloudTrail for the attachment failure: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTransitGatewayVpcAttachment --max-results 10`
- For cross-account attachments, verify RAM share acceptance: `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- Verify the TGW is in available state: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id> --query 'TransitGateways[*].State'`

MAY:
- Check if auto-accept is enabled for shared attachments
- Verify no SCP blocks the attachment creation

## Phase 2 — Remediate

MUST:
- Accept pending cross-account attachments: `aws ec2 accept-transit-gateway-vpc-attachment --transit-gateway-attachment-id <attach-id>`
- Fix IAM permissions for attachment creation
- Ensure TGW is in available state before creating attachments

SHOULD:
- Create VPC attachment with subnets in all needed AZs:
  `aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id <tgw-id> --vpc-id <vpc-id> --subnet-ids <subnet-ids>`
- Enable appliance mode if routing through a firewall VPC

MAY:
- Request quota increase for attachments per TGW
- Enable auto-accept on the TGW for shared attachments

## Common Issues

- symptoms: "Attachment stuck in pendingAcceptance"
  diagnosis: "Cross-account attachment requires acceptance by TGW owner."
  resolution: "TGW owner must accept: aws ec2 accept-transit-gateway-vpc-attachment."

- symptoms: "Attachment in failed state"
  diagnosis: "Subnet or VPC configuration issue, or quota exceeded."
  resolution: "Check error message in attachment details. Fix subnet/VPC issues or request quota increase."

- symptoms: "DuplicateTransitGatewayAttachment error"
  diagnosis: "A VPC can only have one attachment to a given TGW."
  resolution: "Modify the existing attachment instead of creating a new one."

## Output Format

```yaml
root_cause: "attachment_failure — <specific_cause>"
evidence:
  - type: attachment_state
    content: "<attachment status and error>"
  - type: iam_permissions
    content: "<IAM evaluation>"
  - type: quota_check
    content: "<attachment count vs quota>"
severity: HIGH
mitigation:
  immediate: "Fix the specific attachment blocker"
  long_term: "Automate attachment lifecycle with IaC"
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
