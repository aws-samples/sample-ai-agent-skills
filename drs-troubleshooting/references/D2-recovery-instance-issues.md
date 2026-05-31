---
title: "D2 — Recovery Instance Issues"
description: "Diagnose recovery instance post-launch problems"
status: active
severity: HIGH
triggers:
  - "recovery instance not working"
  - "instance launched but not accessible"
  - "recovery instance boot failure"
owner: devops-agent
objective: "Resolve recovery instance post-launch issues"
context: "After successful launch, recovery instances may have OS-level boot issues, network configuration problems, or application failures. These are distinct from DRS launch failures."
---

## Phase 1 — Triage

MUST:
- Check recovery instance status: `aws drs describe-recovery-instances --filters 'sourceServerIDs=<id>'`
- Check EC2 instance status checks: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Get console output: `aws ec2 get-console-output --instance-id <instance-id>`

SHOULD:
- Check EC2 system log for boot errors
- Verify network configuration (VPC, subnet, security groups)
- Check if instance has public/private IP as expected

MAY:
- Use EC2 Serial Console for debugging boot issues
- Check EBS volume attachment status

## Phase 2 — Remediate

MUST:
- Address OS-level boot issues (driver, bootloader, network config)
- Verify security groups allow required application traffic

SHOULD:
- Install required drivers if missing (e.g., ENA, NVMe)
- Update network configuration for AWS environment

MAY:
- Use post-launch actions to automate OS configuration changes

## Common Issues

- symptoms: "Instance running but failing status checks"
  diagnosis: "OS boot failure — check console output for kernel panic or driver issues."
  resolution: "Fix boot configuration. May need to mount volume on helper instance."

## Output Format

```yaml
root_cause: "recovery_instance — <specific_cause>"
evidence:
  - type: instance_status
    content: "<EC2 status checks and console output>"
severity: HIGH
mitigation:
  immediate: "Fix OS-level or network issue"
  long_term: "Add post-launch actions to automate OS configuration"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Recovery instances contain production data |
| Blast Radius | HIGH — Recovery instance failures mean workloads are not recovered |
| Rollback Complexity | MEDIUM — May need to relaunch or fix OS-level issues |
| Compliance Scope | CRITICAL — Recovery instance availability is the DR objective |

## Escalation Conditions

- Escalate to operations if recovery instances fail status checks
- Escalate to application teams if instances launch but applications don't start
- Escalate to AWS Support if OS-level boot issues cannot be resolved

## Data Sensitivity

- **Recovery instances** contain production data — apply full security controls
- **Console output** may contain sensitive boot information
- **EBS volumes** contain replicated production data

## PROHIBITED Actions

- Do NOT leave recovery instances running without security controls (security groups, encryption)
- Do NOT access recovery instance data without authorization
- Do NOT terminate recovery instances during actual DR without management approval

## Phase 3 — Rollback

- If recovery instance is non-functional, terminate and relaunch from replicated data
- Use post-launch actions to automate OS configuration fixes
- Verify application functionality before routing production traffic

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
