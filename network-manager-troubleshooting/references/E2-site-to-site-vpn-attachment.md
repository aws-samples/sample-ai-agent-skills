---
title: "E2 — Site-to-Site VPN Attachment"
description: "Diagnose site-to-site VPN attachment creation and connectivity issues in core networks"
status: active
severity: HIGH
triggers: ["VPN attachment failed", "VPN attachment error", "site-to-site VPN core network", "VPN tunnel down on core network"]
owner: devops-agent
objective: "Resolve site-to-site VPN attachment issues in AWS Cloud WAN core networks"
context: "Site-to-site VPN attachments connect on-premises networks to core network segments. The VPN attachment state being AVAILABLE does not mean tunnels are UP."
---

## Phase 1 — Triage

MUST:
- List VPN attachments: `aws networkmanager list-attachments --core-network-id <id> --attachment-type SITE_TO_SITE_VPN`
- Get VPN attachment details: `aws networkmanager get-site-to-site-vpn-attachment --attachment-id <id>`
- Check VPN connection tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`

SHOULD:
- Verify the customer gateway configuration matches the VPN connection parameters
- Check if the VPN attachment is assigned to the correct segment
- Review the core network policy attachment-policies for VPN acceptance rules

MAY:
- Check CloudWatch metrics for tunnel status: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id>`
- Review VPN connection configuration download for customer gateway setup

## Phase 2 — Remediate

MUST:
- For REJECTED VPN attachment: update tags to match attachment-policy conditions and recreate
- For tunnel DOWN: verify customer gateway configuration (IKE, IPsec, BGP parameters)
- Ensure the VPN connection's target is the core network (not a VGW or TGW directly)

SHOULD:
- Accept the VPN attachment if manual acceptance is required: `aws networkmanager accept-attachment --attachment-id <id>`
- Verify BGP session establishment if using dynamic routing

MAY:
- Enable VPN logging for detailed tunnel diagnostics
- Configure both tunnels for redundancy

## Common Issues

- symptoms: "VPN attachment AVAILABLE but both tunnels DOWN"
  diagnosis: "Customer gateway not configured or IKE/IPsec parameters mismatch."
  resolution: "Download the VPN configuration and apply it to the customer gateway. Verify IKE version, encryption, and authentication parameters."

- symptoms: "VPN attachment REJECTED by core network"
  diagnosis: "VPN connection tags do not match any attachment-policy condition."
  resolution: "Tag the VPN connection to match an attachment-policy condition, delete the rejected attachment, and recreate."

## Safety Ratings

```
safety_ratings:
  - "list-attachments: GREEN — read-only enumeration"
  - "get-site-to-site-vpn-attachment: GREEN — read-only detail retrieval"
  - "describe-vpn-connections: GREEN — read-only VPN status"
  - "create-site-to-site-vpn-attachment: YELLOW — creates VPN attachment to core network"
  - "delete-attachment: RED — removes VPN connectivity to core network"
```

## Escalation Conditions

- VPN tunnels remain DOWN after customer gateway configuration is verified
- VPN attachment creation fails with internal errors
- BGP session flapping causing route instability in the core network segment
- VPN attachment affects production on-premises connectivity

## Data Sensitivity

- `aws networkmanager get-site-to-site-vpn-attachment`: MEDIUM — reveals VPN configuration
- `aws ec2 describe-vpn-connections`: HIGH — reveals tunnel endpoints, PSK references, and CGW details
- `aws cloudwatch get-metric-statistics (VPN)`: LOW — reveals tunnel up/down status

## Prohibited Actions

- NEVER delete a production VPN attachment without confirming backup connectivity exists
- NEVER share VPN pre-shared keys in logs, tickets, or chat
- NEVER modify VPN connection parameters on a live production tunnel without a maintenance window
- NEVER assume both tunnels are UP just because the attachment is AVAILABLE

## Phase 3 — Rollback

1. If a VPN attachment was deleted in error, recreate it: `aws networkmanager create-site-to-site-vpn-attachment --core-network-id <id> --vpn-connection-arn <arn> --tags Key=segment,Value=<segment>`
2. Accept the new attachment if required: `aws networkmanager accept-attachment --attachment-id <new-id>`
3. Verify tunnel status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id>`
4. Verify routes in the segment include on-premises prefixes
5. Confirm end-to-end connectivity from on-premises to cloud workloads

## Output Format

```yaml
root_cause: "vpn_attachment — <specific_cause>"
evidence:
  - type: vpn_state
    content: "<attachment and tunnel details>"
severity: HIGH
mitigation:
  immediate: "Fix VPN attachment or tunnel configuration"
  long_term: "Implement VPN monitoring with CloudWatch alarms"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
