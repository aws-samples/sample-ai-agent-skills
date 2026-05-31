---
title: "H1 — Active Directory Integration Issues"
description: "Diagnose Active Directory join and authentication problems for SMB shares"
status: active
severity: HIGH
triggers:
  - "AD join failed"
  - "Active Directory"
  - "domain join error"
  - "SMB authentication failed"
owner: devops-agent
objective: "Identify and resolve Active Directory integration issues for SMB file shares"
context: "SMB file shares on file gateways can use Active Directory for authentication. The gateway must join the AD domain. Requirements include DNS resolution of the AD domain, network connectivity to domain controllers on specific ports, time synchronization within 5 minutes, and valid domain join credentials."
---

## Phase 1 — Triage

MUST:
- Check SMB settings and AD status: `aws storagegateway describe-smb-settings --gateway-arn <gateway-arn>`
- Verify DNS resolves the AD domain name from the gateway
- Test connectivity to AD domain controllers on ports 389, 445, 88, 53
- Check time synchronization between gateway and AD (must be within 5 minutes)

SHOULD:
- Verify domain join credentials have sufficient permissions
- Check if the gateway computer account exists in AD
- Review gateway logs for AD-related errors

MAY:
- Check AD group policies that may affect the gateway
- Verify AD site and services configuration

## Phase 2 — Remediate

MUST:
- Configure DNS on gateway to resolve AD domain
- Ensure network connectivity to AD domain controllers
- Sync NTP on gateway VM
- Join domain: `aws storagegateway join-domain --gateway-arn <arn> --domain-name <domain> --user-name <user> --password <password>`

SHOULD:
- Use a dedicated service account for domain join
- Specify organizational unit: `--organizational-unit "OU=StorageGateways,DC=example,DC=com"`

MAY:
- Configure AD group-based access control on SMB shares
- Set up monitoring for AD connectivity

## Common Issues

- symptoms: "Domain join fails with DNS error"
  diagnosis: "Gateway cannot resolve AD domain name."
  resolution: "Configure gateway DNS to use AD DNS servers or DNS that forwards to AD."

- symptoms: "Domain join fails with authentication error"
  diagnosis: "Invalid credentials or insufficient permissions."
  resolution: "Verify credentials. User needs permission to join computers to domain."

- symptoms: "Domain join fails with time sync error"
  diagnosis: "Time difference between gateway and AD exceeds 5 minutes."
  resolution: "Configure NTP on gateway. Sync with AD domain controller or reliable NTP source."

## Output Format

```yaml
root_cause: "ad_integration — <specific_cause>"
evidence:
  - type: ad_status
    content: "<SMB settings and domain join status>"
  - type: dns_resolution
    content: "<AD domain DNS resolution>"
  - type: connectivity
    content: "<AD port connectivity tests>"
severity: HIGH
mitigation:
  immediate: "Fix DNS, connectivity, or time sync and retry domain join"
  long_term: "Monitor AD connectivity and time synchronization"
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
