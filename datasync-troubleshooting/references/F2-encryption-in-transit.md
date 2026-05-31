---
title: "F2 — Encryption in Transit Issues"
description: "Diagnose encryption in transit configuration for DataSync transfers"
status: active
severity: MEDIUM
triggers:
  - "encryption in transit"
  - "TLS configuration"
  - "data not encrypted"
  - "security compliance"
owner: devops-agent
objective: "Verify and configure encryption in transit for DataSync transfers"
context: "DataSync encrypts data in transit by default using TLS 1.2 between the agent and AWS. For NFS/SMB locations, traffic between the agent and on-premises storage may not be encrypted by default. Understanding which segments are encrypted is critical for compliance."
---

## Phase 1 — Triage

MUST:
- Verify task security configuration: `aws datasync describe-task --task-arn <task-arn> --query 'Options.SecurityDescriptorCopyFlags'`
- Identify all network segments in the transfer path
- Determine which segments require encryption for compliance
- Check if VPC endpoints are used (traffic stays within AWS network)

SHOULD:
- Verify agent-to-AWS communication uses TLS (default, always on)
- Check NFS/SMB mount options for encryption settings
- Review compliance requirements for data in transit

MAY:
- Check if AWS PrivateLink is used for DataSync service endpoint
- Verify certificate validation settings

## Phase 2 — Remediate

MUST:
- Understand that agent-to-AWS traffic is always encrypted with TLS 1.2
- For NFS: consider NFSv4 with Kerberos for encryption between agent and NFS server
- For SMB: enable SMB encryption between agent and SMB server

SHOULD:
- Use VPC endpoints for DataSync to keep traffic within AWS network: `aws ec2 create-vpc-endpoint --vpc-id <vpc-id> --service-name com.amazonaws.<region>.datasync`
- Document which segments are encrypted and which are not

MAY:
- Implement network-level encryption (IPsec VPN) for agent-to-storage traffic
- Use AWS Direct Connect with MACsec for physical link encryption

## Common Issues

- symptoms: "Compliance audit flags unencrypted data transfer"
  diagnosis: "Agent-to-storage traffic (NFS/SMB) may not be encrypted."
  resolution: "Agent-to-AWS is always TLS encrypted. For agent-to-storage, use NFSv4+Kerberos or SMB encryption."

- symptoms: "Data traversing public internet"
  diagnosis: "Agent communicating with DataSync over public endpoints."
  resolution: "Use VPC endpoints for DataSync to keep traffic within AWS network."

- symptoms: "TLS certificate errors"
  diagnosis: "Agent cannot validate DataSync service TLS certificate."
  resolution: "Ensure agent has correct time (NTP). Check proxy/firewall TLS inspection."

## Output Format

```yaml
root_cause: "encryption_in_transit — <specific_cause>"
evidence:
  - type: transfer_path
    content: "<network segments and encryption status>"
  - type: task_config
    content: "<security configuration>"
  - type: endpoint_type
    content: "<public or VPC endpoint>"
severity: MEDIUM
mitigation:
  immediate: "Identify and address unencrypted segments"
  long_term: "Implement end-to-end encryption for all transfer paths"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Encryption in transit protects data during transfer |
| Blast Radius | LOW — Encryption configuration does not affect data content |
| Rollback Complexity | LOW — Configuration changes only |
| Compliance Scope | CRITICAL — Encryption in transit is a compliance requirement for most frameworks |

## Escalation Conditions

- Escalate IMMEDIATELY to security if data is being transferred without encryption on any segment
- Escalate to compliance if encryption gaps are discovered during audit
- Escalate to network team if VPC endpoint configuration is needed

## Data Sensitivity

- **Transfer path analysis** reveals network architecture — handle appropriately
- **Encryption status per segment** is security-critical information
- **VPC endpoint configuration** is infrastructure data

## PROHIBITED Actions

- Do NOT transfer sensitive data over unencrypted network segments
- Do NOT disable TLS inspection on firewalls without understanding the security impact
- Do NOT assume all segments are encrypted — verify each segment independently
- Do NOT skip encryption compliance verification for new transfer paths

## Phase 3 — Rollback

- Restore VPC endpoint configuration if it was modified
- Revert NFS/SMB encryption settings if they were changed
- Re-verify encryption status on all transfer path segments
- Document encryption compliance status for audit records

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
