---
title: "B2 — Launch Template Errors"
description: "Diagnose launch template configuration issues causing scaling failures"
status: active
severity: HIGH
triggers:
  - "launch template error"
  - "invalid launch template"
  - "template version not found"
  - "launch configuration error"
  - "template parameter invalid"
owner: devops-agent
objective: "Identify and fix launch template configuration errors"
context: "Launch template errors include: referencing deleted AMIs, invalid instance types, missing security groups, incorrect IAM profiles, invalid user data, deleted key pairs, or incompatible EBS configurations. Launch templates support versioning — the ASG may reference a specific version, $Latest, or $Default."
---

## Phase 1 — Triage

MUST:
- Check which template version ASG uses: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].{Template:LaunchTemplate}'`
- Describe the template version: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id> --versions <version>`
- Validate AMI exists: `aws ec2 describe-images --image-ids <ami-id> --query 'Images[*].{Id:ImageId,State:State}'`
- Check security groups exist: `aws ec2 describe-security-groups --group-ids <sg-ids>`

SHOULD:
- Compare $Latest vs $Default version: `aws ec2 describe-launch-template-versions --launch-template-id <lt-id> --versions '$Latest' '$Default'`
- Verify instance profile exists: `aws iam get-instance-profile --instance-profile-name <name>`
- Check user data encoding (must be base64)
- Verify EBS volume configuration compatibility with instance type

MAY:
- Review template version history for recent changes
- Check if template was modified by IaC (CloudFormation/Terraform)

## Phase 2 — Remediate

MUST:
- Create a new template version with corrected configuration
- Update ASG to use the corrected version
- Verify launch succeeds with the new version

SHOULD:
- Create corrected version: `aws ec2 create-launch-template-version --launch-template-id <lt-id> --source-version <version> --launch-template-data '{"ImageId":"<new-ami>"}'`
- Update ASG: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --launch-template LaunchTemplateId=<lt-id>,Version='$Latest'`

MAY:
- Set the corrected version as default
- Implement template validation in CI/CD pipeline

## Output Format

```yaml
root_cause: "launch_template_error — <specific_cause>"
evidence:
  - type: template_config
    content: "<template version details>"
  - type: resource_validation
    content: "<resource existence check>"
severity: HIGH
mitigation:
  immediate: "Create corrected launch template version"
  long_term: "Implement template validation and version management"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create new template version with corrected config | YELLOW | New version — test before applying to production ASG |
| Update ASG to use corrected version | YELLOW | Changes instance configuration for new launches |
| Verify launch succeeds with new version | GREEN | Testing — non-destructive |
| Set corrected version as default | YELLOW | Affects all ASGs using $Default |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Template version changes affect all new instance launches
- Setting default version affects all ASGs referencing $Default

## Data Sensitivity

- Launch template contains instance configuration including user data, IAM profiles, and security groups
- Template version history reveals infrastructure evolution
- User data may contain bootstrap scripts with sensitive configuration

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER delete launch template versions that are actively referenced by ASGs
- NEVER modify user data without understanding the bootstrap process

## Phase 3 — Rollback

- If new template version is faulty: update ASG to reference the previous working version
- If default version was changed: restore previous default using `aws ec2 modify-launch-template --default-version`
- If ASG was updated to wrong version: revert ASG launch template version reference
- Document all template version changes for operational audit trail

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
