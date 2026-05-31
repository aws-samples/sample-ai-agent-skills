---
title: "E1 — Direct Connect Gateway Attachment Issues"
description: "Diagnose DX gateway attachment to Transit Gateway problems"
status: active
severity: HIGH
triggers:
  - "DX gateway attachment failed"
  - "Direct Connect TGW not working"
  - "DX gateway association"
  - "on-premises to TGW via DX"
owner: devops-agent
objective: "Identify and resolve DX gateway to TGW attachment issues"
context: "Direct Connect gateways connect to TGWs via transit gateway attachments. The DX gateway must be associated with the TGW. Allowed prefixes control which CIDRs are advertised. DX gateway and TGW can be in different accounts."
---

## Phase 1 — Triage

MUST:
- Check DX gateway associations: `aws directconnect describe-direct-connect-gateway-associations --direct-connect-gateway-id <dx-gw-id>`
- Verify the TGW attachment for DX: `aws ec2 describe-transit-gateway-attachments --filters Name=resource-type,Values=direct-connect-gateway Name=transit-gateway-id,Values=<tgw-id>`
- Check allowed prefixes on the association: `aws directconnect describe-direct-connect-gateway-associations --direct-connect-gateway-id <dx-gw-id> --query 'directConnectGatewayAssociations[*].allowedPrefixesToDirectConnectGateway'`
- Verify the DX gateway state: `aws directconnect describe-direct-connect-gateways --direct-connect-gateway-id <dx-gw-id>`

SHOULD:
- Check if the DX gateway ASN conflicts with the TGW ASN
- Verify the virtual interface is attached to the DX gateway
- Check TGW route table for DX-learned routes

MAY:
- Check CloudTrail for DX gateway association events
- Verify cross-account permissions if DX gateway and TGW are in different accounts

## Phase 2 — Remediate

MUST:
- Create or fix the DX gateway to TGW association:
  `aws directconnect create-direct-connect-gateway-association --direct-connect-gateway-id <dx-gw-id> --gateway-id <tgw-id> --add-allowed-prefixes-to-direct-connect-gateway cidr=<cidr>`
- Configure allowed prefixes to include all CIDRs that should be advertised to on-premises
- Enable route propagation from the DX attachment in the TGW route table

SHOULD:
- Ensure ASNs are unique between DX gateway, TGW, and on-premises
- Verify transit VIF is properly configured on the DX connection

MAY:
- Set up redundant DX connections for high availability
- Monitor DX gateway metrics

## Common Issues

- symptoms: "DX gateway association stuck in associating state"
  diagnosis: "ASN conflict or cross-account permission issue."
  resolution: "Ensure unique ASNs. For cross-account, accept the association proposal."

- symptoms: "On-premises cannot reach VPCs through TGW via DX"
  diagnosis: "Allowed prefixes not configured or TGW route table missing routes."
  resolution: "Add VPC CIDRs to allowed prefixes and enable propagation in TGW route table."

- symptoms: "Only some VPC CIDRs reachable from on-premises"
  diagnosis: "Allowed prefixes list incomplete."
  resolution: "Add all required CIDRs to the allowed prefixes on the DX gateway association."

## Output Format

```yaml
root_cause: "dx_gateway_attachment_issue — <specific_cause>"
evidence:
  - type: dx_association
    content: "<DX gateway association status>"
  - type: allowed_prefixes
    content: "<configured allowed prefixes>"
  - type: tgw_routes
    content: "<routes from DX in TGW route table>"
severity: HIGH
mitigation:
  immediate: "Fix DX gateway association and allowed prefixes"
  long_term: "Automate DX gateway management with IaC"
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
