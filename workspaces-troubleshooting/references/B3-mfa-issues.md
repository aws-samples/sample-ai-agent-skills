---
title: "B3 — MFA Issues"
description: "Diagnose multi-factor authentication failures for WorkSpaces"
status: active
severity: HIGH
triggers:
  - "MFA not working"
  - "RADIUS authentication failed"
  - "two-factor auth error"
  - "MFA timeout"
owner: devops-agent
objective: "Resolve MFA configuration and authentication issues for WorkSpaces"
context: "MFA for WorkSpaces depends on directory type. AD Connector supports RADIUS only. AWS Managed Microsoft AD supports RADIUS and smart card. Simple AD does not support MFA. RADIUS server must be reachable from directory subnets."
---

## Phase 1 — Triage

MUST:
- Check directory type and MFA status: `aws ds describe-directories --directory-ids <dir-id> --query 'DirectoryDescriptions[].{Type:Type,RadiusStatus:RadiusStatus}'`
- Verify RADIUS settings: `aws ds describe-directories --directory-ids <dir-id> --query 'DirectoryDescriptions[].RadiusSettings'`
- Check security group allows traffic to RADIUS server (port 1812 UDP): `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify RADIUS server is reachable from directory subnets

SHOULD:
- Check RADIUS server logs for authentication attempts
- Verify shared secret matches between directory and RADIUS server
- Test RADIUS connectivity from an instance in the same subnet

MAY:
- Check if RADIUS server has rate limiting that blocks bulk authentication
- Verify NTP synchronization between RADIUS server and domain controllers

## Phase 2 — Remediate

MUST:
- Ensure RADIUS server is reachable on port 1812 UDP from directory subnets
- Verify shared secret matches on both sides
- Confirm directory type supports MFA (not Simple AD)

SHOULD:
- Configure RADIUS timeout to at least 20 seconds
- Set up redundant RADIUS servers for high availability
- Enable RADIUS server logging for troubleshooting

MAY:
- Consider AWS Managed Microsoft AD with smart card for certificate-based MFA

## Common Issues

- symptoms: "RADIUS status shows Failed"
  diagnosis: "Directory cannot reach RADIUS server or shared secret mismatch."
  resolution: "Verify network connectivity on port 1812 UDP. Confirm shared secret matches."

- symptoms: "MFA prompt not appearing"
  diagnosis: "MFA not enabled on the directory or RADIUS not configured."
  resolution: "Enable MFA and configure RADIUS settings on the directory."

- symptoms: "MFA works for some users but not others"
  diagnosis: "RADIUS server user enrollment incomplete or user-specific RADIUS policy."
  resolution: "Verify all users are enrolled in the MFA solution on the RADIUS server."

## Output Format

```yaml
root_cause: "mfa_issue — <specific_cause>"
evidence:
  - type: radius_status
    content: "<RADIUS configuration and status>"
  - type: network_connectivity
    content: "<security group rules and RADIUS reachability>"
severity: HIGH
mitigation:
  immediate: "Fix RADIUS connectivity, shared secret, or enrollment"
  long_term: "Set up redundant RADIUS servers and monitoring"
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
