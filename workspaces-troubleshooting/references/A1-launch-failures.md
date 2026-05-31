---
title: "A1 — Launch Failures"
description: "Diagnose why a WorkSpace fails to launch or stays in PENDING state"
status: active
severity: HIGH
triggers:
  - "workspace launch failed"
  - "workspace stuck in pending"
  - "cannot create workspace"
  - "workspace error state"
owner: devops-agent
objective: "Identify and resolve WorkSpace provisioning failures"
context: "WorkSpace launch failures are caused by insufficient subnet IPs, directory issues, bundle incompatibility, IAM permission problems, or service quota limits. WorkSpaces require two subnets in different AZs with available IP addresses."
---

## Phase 1 — Triage

MUST:
- Check WorkSpace state and error code: `aws workspaces describe-workspaces --workspace-id <id>`
- Verify directory status: `aws workspaces describe-workspace-directories --directory-ids <dir-id>`
- Check available IPs in registered subnets: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2> --query 'Subnets[].{SubnetId:SubnetId,AvailableIps:AvailableIpAddressCount,AZ:AvailabilityZone}'`
- Verify bundle exists and is available: `aws workspaces describe-workspace-bundles --bundle-ids <bundle-id>`

SHOULD:
- Check WorkSpaces service quota: `aws service-quotas get-service-quota --service-code workspaces --quota-code L-34278094`
- Review CloudTrail for CreateWorkspaces errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateWorkspaces --max-results 10`
- Verify the user exists in the directory: `aws ds describe-directories --directory-ids <dir-id>`

MAY:
- Check if the target AZ has capacity issues
- Verify custom image status if using custom bundle: `aws workspaces describe-workspace-images`

## Phase 2 — Remediate

MUST:
- Ensure subnets have sufficient available IP addresses (minimum 2 per WorkSpace)
- Fix directory registration if directory is not in Active state
- Use a valid bundle ID that matches the directory region

SHOULD:
- Request quota increase if at limit
- Retry launch in a different AZ if capacity constrained
- Verify user account is enabled in Active Directory

MAY:
- Create a new custom bundle if existing one is corrupted

## Common Issues

- symptoms: "WorkSpace stuck in PENDING for over 20 minutes"
  diagnosis: "Subnet has no available IP addresses or directory is not in Active state."
  resolution: "Check subnet available IPs. Verify directory status. Terminate and relaunch if stuck."

- symptoms: "ERROR state immediately after launch"
  diagnosis: "Bundle incompatible with directory type or user not found in directory."
  resolution: "Verify bundle compatibility. Confirm user exists in the registered directory."

- symptoms: "Quota exceeded error"
  diagnosis: "Account has reached the WorkSpaces service quota for the region."
  resolution: "Request quota increase via Service Quotas console or terminate unused WorkSpaces."

## Output Format

```yaml
root_cause: "launch_failure — <specific_cause>"
evidence:
  - type: workspace_state
    content: "<workspace status and error details>"
  - type: subnet_capacity
    content: "<available IPs in registered subnets>"
severity: HIGH
mitigation:
  immediate: "Fix subnet capacity, directory status, or bundle configuration"
  long_term: "Monitor subnet IP utilization and set up quota alarms"
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
