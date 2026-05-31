---
title: "A4 — Launch Template / Configuration Errors"
description: "Diagnose EC2 launch failures caused by invalid launch template or configuration parameters"
status: active
severity: MEDIUM
triggers:
  - "InvalidParameterCombination"
  - "InvalidBlockDeviceMapping"
  - "InvalidSubnetID"
  - "InvalidGroup"
  - "Unsupported"
owner: devops-agent
objective: "Identify the configuration error and fix the launch template or parameters"
context: "Launch templates can contain stale references (deleted subnets, SGs, KMS keys), incompatible parameter combinations (instance type vs EBS type, placement group constraints), or missing required parameters."
---

## Phase 1 — Triage

MUST:
- Get the exact error from CloudTrail RunInstances event
- Review the launch template: `aws ec2 describe-launch-template-versions --launch-template-id <id>`
- Validate all referenced resources exist (subnet, SG, key pair, IAM profile, AMI)
- Check parameter compatibility (instance type supports requested features)

SHOULD:
- Verify subnet has available IP addresses
- Check placement group constraints if specified
- Verify EBS volume type is supported in the target AZ

MAY:
- Compare with a previously successful launch template version

## Common Issues

- symptoms: "InvalidParameterCombination for EBS volume"
  diagnosis: "io2 Block Express volumes require Nitro instances. gp2/gp3 max size is 16 TiB."
  resolution: "Match EBS volume type to instance capabilities."

- symptoms: "InvalidSubnetID.NotFound"
  diagnosis: "Subnet referenced in launch template was deleted."
  resolution: "Update launch template with valid subnet ID."

- symptoms: "InvalidGroup.NotFound"
  diagnosis: "Security group referenced in launch template was deleted or is in a different VPC."
  resolution: "Update launch template with valid security group in the same VPC as the subnet."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-launch-template-versions to review template: GREEN — read-only"
  - "Create new launch template version with fixes: GREEN — does not affect existing version"
  - "Update default launch template version: YELLOW — affects future launches from this template"
  - "Modify subnet or security group references: YELLOW — recoverable by reverting template version"
  - "Delete launch template versions: RED — version permanently removed"
```

## Escalation Conditions
- Launch template is used by production Auto Scaling groups
- Template references cross-account resources (shared subnets, security groups)
- Multiple launch templates are affected by the same stale reference
- Fix requires modifying VPC infrastructure (subnets, security groups, route tables)
- Template is managed by IaC (CloudFormation/Terraform) and manual changes would cause drift

## Data Sensitivity
- HIGH: describe-launch-template-versions (may contain user data scripts with secrets, IAM profiles, key pairs)
- MEDIUM: describe-subnets, describe-security-groups (network topology information)
- LOW: describe-instance-type-offerings (public data)

## Prohibited Actions
- NEVER suggest deleting a launch template that is actively referenced by an ASG or Spot Fleet
- NEVER suggest modifying security groups inline without understanding current rules
- NEVER suggest changing subnet assignments without verifying AZ compatibility
- NEVER suggest removing encryption settings from block device mappings

## Phase 3 — Rollback
- If launch template version was updated: set default version back to previous version number
- If security group was modified: revert security group rules to previous state
- If subnet reference was changed: update template to point back to original subnet
- If new template was created: delete the new template and revert ASG/Fleet to use original

## Output Format

```yaml
root_cause: "<invalid_parameter|stale_reference|incompatible_config> — <detail>"
evidence:
  - type: launch_template
    content: "<problematic parameter from template>"
severity: MEDIUM
mitigation:
  immediate: "Fix the specific parameter in launch template"
  long_term: "Validate launch templates before deployment, use IaC with dependency tracking"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
