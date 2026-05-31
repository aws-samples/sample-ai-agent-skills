---
title: "F2 — Firewall Port Configuration"
description: "Diagnose firewall and port configuration issues for Storage Gateway"
status: active
severity: HIGH
triggers:
  - "firewall blocking"
  - "port configuration"
  - "required ports"
  - "port not open"
owner: devops-agent
objective: "Identify and resolve firewall port configuration issues for Storage Gateway"
context: "Storage Gateway requires specific ports for different functions: 443 (AWS communication), 80 (activation only), 3260 (iSCSI for volume/tape), 2049 (NFS for file gateway), 445 (SMB for file gateway), 1026-1028/1031/2222 (local console). Missing port rules cause specific feature failures."
---

## Phase 1 — Triage

MUST:
- Identify gateway type and required ports
- Check security groups for all required inbound/outbound rules
- Check NACLs for port restrictions
- Test connectivity on each required port

SHOULD:
- Check on-premises firewall rules if gateway is on-premises
- Verify both inbound and outbound rules (NACLs are stateless)
- Check for port conflicts with other services

MAY:
- Review firewall logs for blocked connections
- Check if deep packet inspection is interfering with TLS

## Phase 2 — Remediate

MUST:
- Open required outbound ports: TCP 443 (AWS communication)
- For file gateway: open inbound TCP 2049 (NFS) or TCP 445 (SMB) from clients
- For volume/tape gateway: open inbound TCP 3260 (iSCSI) from initiators
- For activation: temporarily open inbound TCP 80

SHOULD:
- Open local console ports: TCP 1026-1028, 1031, 2222 (from admin network only)
- Restrict inbound ports to specific client IP ranges
- Document all required ports for the gateway type

MAY:
- Use VPC endpoints to avoid internet-facing firewall rules
- Implement network segmentation for gateway traffic

## Common Issues

- symptoms: "iSCSI initiator cannot connect to volume/tape gateway"
  diagnosis: "Port 3260 not open inbound on gateway security group."
  resolution: "Add inbound rule for TCP 3260 from iSCSI initiator IP range."

- symptoms: "NFS mount fails from client"
  diagnosis: "Port 2049 not open inbound on gateway security group."
  resolution: "Add inbound rule for TCP 2049 from NFS client IP range."

- symptoms: "Gateway activated but cannot communicate with AWS"
  diagnosis: "Port 443 outbound blocked after activation."
  resolution: "Ensure outbound TCP 443 is permanently open (not just during activation)."

## Output Format

```yaml
root_cause: "firewall_ports — <specific_cause>"
evidence:
  - type: required_ports
    content: "<ports needed for gateway type>"
  - type: security_groups
    content: "<current inbound/outbound rules>"
  - type: connectivity_tests
    content: "<port connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Open required ports in security groups and firewalls"
  long_term: "Document port requirements and automate security group configuration"
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

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
