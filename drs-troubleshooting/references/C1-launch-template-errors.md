---
title: "C1 — Launch Template Errors"
description: "Diagnose launch configuration and template issues"
status: active
severity: HIGH
triggers:
  - "launch failed"
  - "launch template error"
  - "cannot launch recovery"
owner: devops-agent
objective: "Resolve launch configuration issues preventing recovery"
context: "DRS uses launch configurations to define how recovery instances are created. Issues include invalid instance types, subnet misconfigurations, security group problems, or IAM role issues."
---

## Phase 1 — Triage

MUST:
- Get launch config: `aws drs get-launch-configuration --source-server-id <id>`
- Verify target instance type is valid in the target region/AZ
- Check target subnet exists and has available IPs
- Verify security groups are valid

SHOULD:
- Check IAM instance profile specified in launch config
- Verify EBS volume types are available in target AZ
- Review launch configuration template: `aws drs describe-launch-configuration-templates`

MAY:
- Check EC2 service quotas for the target instance type
- Verify AMI compatibility if custom AMI is specified

## Phase 2 — Remediate

MUST:
- Fix invalid instance type, subnet, or security group references
- Update launch configuration: `aws drs update-launch-configuration --source-server-id <id> --target-instance-type-right-sizing-method BASIC`

SHOULD:
- Validate all launch config parameters before attempting recovery
- Ensure target subnet has sufficient IP addresses

MAY:
- Use launch configuration templates for consistent settings across servers

## Common Issues

- symptoms: "Launch fails with InsufficientInstanceCapacity"
  diagnosis: "Target instance type not available in the specified AZ."
  resolution: "Change instance type or target subnet to a different AZ."

## Output Format

```yaml
root_cause: "launch_template — <specific_cause>"
evidence:
  - type: launch_config
    content: "<launch configuration details>"
severity: HIGH
mitigation:
  immediate: "Fix the specific launch configuration error"
  long_term: "Validate launch configs as part of DR readiness testing"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Launch configurations reveal DR architecture |
| Blast Radius | HIGH — Invalid launch configs prevent recovery during disasters |
| Rollback Complexity | LOW — Launch configurations can be updated |
| Compliance Scope | HIGH — Valid launch configurations are required for DR readiness |

## Escalation Conditions

- Escalate to DR coordinator if launch configuration errors are found during drills
- Escalate to operations if instance type or subnet issues need resolution

## Data Sensitivity

- **Launch configurations** reveal target infrastructure — operational data
- **IAM instance profiles** define recovery instance permissions

## PROHIBITED Actions

- Do NOT deploy launch configurations without validating through drill tests
- Do NOT use instance types that don't meet application requirements
- Do NOT configure recovery instances in public subnets without security review

## Phase 3 — Rollback

- Restore previous launch configuration settings
- Validate the restored configuration with a drill test

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
