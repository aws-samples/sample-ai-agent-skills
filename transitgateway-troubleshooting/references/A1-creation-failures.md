---
title: "A1 — TGW Creation Failures"
description: "Diagnose why a Transit Gateway creation is failing"
status: active
severity: HIGH
triggers:
  - "TGW creation failed"
  - "cannot create transit gateway"
  - "CreateTransitGateway error"
  - "transit gateway limit"
owner: devops-agent
objective: "Identify why TGW creation is failing and resolve the issue"
context: "TGW creation can fail due to IAM permission issues, service quotas, invalid ASN configuration, or account-level restrictions. The caller needs ec2:CreateTransitGateway permission."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for the caller: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names ec2:CreateTransitGateway`
- Verify TGW quota in the region: `aws service-quotas get-service-quota --service-code vpc --quota-code L-A2478D36`
- List existing TGWs to check current count: `aws ec2 describe-transit-gateways --query 'TransitGateways[?State!=`deleted`]'`
- Check CloudTrail for the creation failure event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTransitGateway --max-results 10`

SHOULD:
- Verify the ASN is valid (64512-65534 for 16-bit, 4200000000-4294967294 for 32-bit): check the requested ASN value
- Confirm the region supports Transit Gateway
- Check for SCP restrictions that may block TGW creation

MAY:
- Review organization-level policies for ec2:CreateTransitGateway denials
- Check if the account has any pending TGW operations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include ec2:CreateTransitGateway and ec2:CreateTags
- Request quota increase if at the TGW limit per region
- Use a valid Amazon-side ASN

SHOULD:
- Create TGW with appropriate options:
  `aws ec2 create-transit-gateway --description "Production TGW" --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=disable,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable,DnsSupport=enable`
- Disable auto-association and auto-propagation for network segmentation

MAY:
- Enable multicast support at creation if needed (cannot be enabled later)
- Tag the TGW for cost allocation and identification

## Common Issues

- symptoms: "UnauthorizedOperation error"
  diagnosis: "IAM policy does not allow ec2:CreateTransitGateway."
  resolution: "Add ec2:CreateTransitGateway permission to the IAM role or user."

- symptoms: "Transit gateway limit exceeded"
  diagnosis: "Account has reached the maximum number of TGWs per region (default 5)."
  resolution: "Request a quota increase via Service Quotas or delete unused TGWs."

- symptoms: "Invalid ASN"
  diagnosis: "The specified Amazon-side ASN is outside the valid range."
  resolution: "Use ASN in range 64512-65534 (16-bit) or 4200000000-4294967294 (32-bit)."

## Output Format

```yaml
root_cause: "tgw_creation_failure — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<IAM policy evaluation result>"
  - type: quota_check
    content: "<current TGW count vs quota>"
  - type: cloudtrail_event
    content: "<error from CloudTrail>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Automate TGW creation with IaC and proper IAM policies"
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
