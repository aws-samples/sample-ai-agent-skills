---
title: "D1 — AD Connector"
description: "Diagnose AD Connector connectivity and authentication issues"
status: active
severity: HIGH
triggers:
  - "AD Connector failed"
  - "directory connector error"
  - "on-premises AD connection"
  - "AD Connector impaired"
owner: devops-agent
objective: "Resolve AD Connector connectivity and configuration issues"
context: "AD Connector proxies directory requests to on-premises Active Directory. It requires network connectivity on ports 53, 88, 389, and 445 to domain controllers. DNS resolution must work for the AD domain name."
---

## Phase 1 — Triage

MUST:
- Check AD Connector status: `aws ds describe-directories --directory-ids <dir-id> --query 'DirectoryDescriptions[].{Status:Stage,Type:Type,DnsIp:DnsIpAddrs}'`
- Verify conditional forwarders: `aws ds describe-conditional-forwarders --directory-id <dir-id>`
- Check security groups allow required ports to on-premises DCs: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify subnet route tables have routes to on-premises network: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Check VPN or Direct Connect status for on-premises connectivity
- Verify DNS resolution of the AD domain from the VPC
- Review directory event topics: `aws ds describe-event-topics --directory-id <dir-id>`

MAY:
- Test port connectivity from an EC2 instance in the same subnet to domain controllers
- Check on-premises firewall logs for blocked connections

## Phase 2 — Remediate

MUST:
- Open ports 53 (DNS), 88 (Kerberos), 389 (LDAP), 445 (SMB) to domain controllers
- Ensure VPN/Direct Connect provides reliable connectivity
- Fix DNS resolution for the AD domain name

SHOULD:
- Configure conditional forwarders for DNS resolution
- Set up redundant connectivity paths (VPN + Direct Connect)
- Monitor AD Connector health with CloudWatch

MAY:
- Consider migrating to AWS Managed Microsoft AD if on-premises connectivity is unreliable

## Common Issues

- symptoms: "AD Connector status is Impaired"
  diagnosis: "Cannot reach domain controllers on required ports."
  resolution: "Verify all four port groups (53, 88, 389, 445) are open. Check VPN/DX connectivity."

- symptoms: "Authentication intermittently fails"
  diagnosis: "Partial port connectivity or DNS resolution failures."
  resolution: "Ensure all ports are open bidirectionally. Verify DNS conditional forwarders."

- symptoms: "AD Connector creation fails"
  diagnosis: "Incorrect domain credentials, unreachable DCs, or DNS failure."
  resolution: "Verify service account credentials. Test DC reachability. Check DNS resolution."

## Output Format

```yaml
root_cause: "ad_connector — <specific_cause>"
evidence:
  - type: directory_status
    content: "<AD Connector state and DNS configuration>"
  - type: network_connectivity
    content: "<port accessibility and route configuration>"
severity: HIGH
mitigation:
  immediate: "Fix port access, DNS resolution, or VPN connectivity"
  long_term: "Set up redundant connectivity and health monitoring"
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

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

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
