---
title: "D2 — AWS Managed AD"
description: "Diagnose AWS Managed Microsoft AD issues for WorkSpaces"
status: active
severity: HIGH
triggers:
  - "managed AD error"
  - "AWS Managed Microsoft AD"
  - "directory service failed"
  - "trust relationship broken"
owner: devops-agent
objective: "Resolve AWS Managed Microsoft AD configuration and operational issues"
context: "AWS Managed Microsoft AD provides a fully managed AD in the AWS cloud. It supports trust relationships with on-premises AD, multi-region replication, and features like Group Policy, LDAPS, and smart card authentication."
---

## Phase 1 — Triage

MUST:
- Check directory status: `aws ds describe-directories --directory-ids <dir-id>`
- Verify trust relationships: `aws ds describe-trusts --directory-id <dir-id>`
- Check domain controller availability: `aws ds describe-domain-controllers --directory-id <dir-id>`
- Verify security group configuration: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Check directory snapshots: `aws ds describe-snapshots --directory-id <dir-id>`
- Verify LDAPS status: `aws ds describe-ldaps-settings --directory-id <dir-id>`
- Review directory sharing status: `aws ds describe-shared-directories --owner-directory-id <dir-id>`

MAY:
- Check schema extensions: `aws ds list-schema-extensions --directory-id <dir-id>`
- Verify conditional forwarders for cross-domain resolution

## Phase 2 — Remediate

MUST:
- Ensure domain controllers are in Active state
- Fix trust relationship if broken (verify trust password and DNS)
- Ensure security groups allow AD traffic between VPC and trusted domains

SHOULD:
- Enable multi-region replication for high availability
- Configure LDAPS for secure directory communication
- Set up directory monitoring with SNS notifications

MAY:
- Restore from snapshot if directory is corrupted
- Add additional domain controllers for performance

## Common Issues

- symptoms: "Trust relationship shows Failed status"
  diagnosis: "DNS resolution between domains failing or trust password mismatch."
  resolution: "Verify conditional forwarders. Reset trust password on both sides."

- symptoms: "Domain controller in Impaired state"
  diagnosis: "Underlying infrastructure issue or security group blocking replication."
  resolution: "Check security groups allow DC-to-DC traffic. AWS may auto-recover the DC."

- symptoms: "Users from trusted domain cannot launch WorkSpaces"
  diagnosis: "Trust relationship not configured or forest trust required instead of external trust."
  resolution: "Verify trust type. Use forest trust for full cross-domain access."

## Output Format

```yaml
root_cause: "managed_ad — <specific_cause>"
evidence:
  - type: directory_status
    content: "<directory and domain controller state>"
  - type: trust_config
    content: "<trust relationship status and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix trust relationship, DC health, or security groups"
  long_term: "Enable multi-region replication and directory monitoring"
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
