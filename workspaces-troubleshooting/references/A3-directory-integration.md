---
title: "A3 — Directory Integration"
description: "Diagnose WorkSpace directory registration and integration failures"
status: active
severity: HIGH
triggers:
  - "directory registration failed"
  - "cannot register directory"
  - "directory not active"
  - "workspace directory error"
owner: devops-agent
objective: "Resolve directory registration and integration issues for WorkSpaces"
context: "WorkSpaces require a registered directory (Simple AD, AD Connector, or AWS Managed Microsoft AD). Registration requires two subnets in different AZs. Directory must be in Active state."
---

## Phase 1 — Triage

MUST:
- Check directory registration status: `aws workspaces describe-workspace-directories`
- Verify directory state: `aws ds describe-directories --directory-ids <dir-id>`
- Confirm subnets are in different AZs: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2> --query 'Subnets[].{SubnetId:SubnetId,AZ:AvailabilityZone}'`
- Check security group allows required traffic: `aws ec2 describe-security-groups --group-ids <sg-id>`

SHOULD:
- Verify DNS resolution from WorkSpaces subnets
- Check if directory is already registered with another WorkSpaces deployment
- Review directory event logs: `aws ds describe-event-topics --directory-id <dir-id>`

MAY:
- Test connectivity to domain controllers from a test instance in the same subnet
- Verify DHCP options set has correct DNS servers

## Phase 2 — Remediate

MUST:
- Ensure directory is in Active state before registration
- Provide two subnets in different AZs during registration
- Configure security group to allow WorkSpaces traffic (ports 4172/4195, 443)

SHOULD:
- Set up DHCP options set with directory DNS servers for the VPC
- Enable directory logging for troubleshooting

MAY:
- Deregister and re-register directory if configuration needs to change

## Common Issues

- symptoms: "Directory registration fails with subnet error"
  diagnosis: "Subnets are in the same AZ or have insufficient available IPs."
  resolution: "Provide two subnets in different AZs with available IP addresses."

- symptoms: "Directory shows Impaired status"
  diagnosis: "Domain controllers are unreachable or experiencing issues."
  resolution: "Check VPC networking, security groups, and domain controller health."

- symptoms: "Users not found when launching WorkSpaces"
  diagnosis: "Directory sync not complete or user account disabled in AD."
  resolution: "Wait for directory sync. Verify user account status in Active Directory."

## Output Format

```yaml
root_cause: "directory_integration — <specific_cause>"
evidence:
  - type: directory_state
    content: "<directory status and configuration>"
  - type: subnet_config
    content: "<subnet AZ and IP availability>"
severity: HIGH
mitigation:
  immediate: "Fix directory state, subnet configuration, or security groups"
  long_term: "Monitor directory health and automate subnet capacity checks"
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
