---
title: "D1 — Notebook Instance Issues"
description: "Diagnose Braket notebook instance creation and access problems"
status: active
severity: MEDIUM
triggers: ["notebook not starting", "braket notebook error", "cannot access notebook", "notebook instance failed"]
owner: devops-agent
objective: "Resolve Braket notebook instance issues"
context: "Braket notebooks are SageMaker notebook instances with Braket SDK pre-installed. Issues may be SageMaker-related."
---

## Phase 1 — Triage

MUST:
- Check notebook instance status: `aws sagemaker describe-notebook-instance --notebook-instance-name <name>`
- Verify IAM role attached to the notebook instance
- Check if the instance type is available in the region

SHOULD:
- Review notebook instance lifecycle configuration if present
- Verify VPC and subnet configuration if using VPC mode
- Check SageMaker service quotas for notebook instances

MAY:
- Review CloudWatch logs for notebook instance startup errors
- Check security group rules if notebook is in a VPC

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions (needs sagemaker:*, braket:*, s3:* at minimum)
- Resolve instance type availability issues
- Fix VPC/subnet/security group configuration if applicable

SHOULD:
- Update Braket SDK in the notebook if outdated: `pip install --upgrade amazon-braket-sdk`
- Verify internet access for package installation (NAT gateway or VPC endpoints)

MAY:
- Use lifecycle configuration scripts to auto-update SDK
- Switch to SageMaker Studio notebooks for a managed experience

## Safety Ratings

```
safety_ratings:
  - "describe-notebook-instance: GREEN — read-only status check"
  - "start-notebook-instance: YELLOW — starts compute instance, incurs charges"
  - "update-notebook-instance: YELLOW — modifies instance configuration"
  - "stop-notebook-instance: YELLOW — stops instance, may interrupt work"
```

## Escalation Conditions

- Notebook instance fails to start repeatedly with InternalFailure
- IAM role is correct but notebook cannot access Braket APIs
- VPC configuration prevents notebook from reaching Braket endpoints
- Notebook instance is in a failed state that cannot be recovered

## Data Sensitivity

- `aws sagemaker describe-notebook-instance`: LOW — returns instance metadata
- Notebook contents: HIGH — may contain proprietary algorithms and research
- IAM role details: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER delete a notebook instance without confirming data is backed up
- NEVER attach AdministratorAccess to the notebook IAM role
- NEVER modify VPC security groups without understanding the impact on other resources
- NEVER ignore lifecycle configuration errors — they can cause persistent startup failures

## Phase 3 — Rollback

1. If notebook configuration was changed, stop and update to previous settings: `aws sagemaker stop-notebook-instance --notebook-instance-name <name>`
2. Restore original IAM role if it was modified
3. Revert VPC/security group changes if they affected other resources
4. If lifecycle configuration was modified, restore the previous version
5. Verify notebook data integrity after any configuration changes

## Common Issues

- symptoms: "Notebook instance stuck in 'Pending' status"
  diagnosis: "Instance type may be unavailable or IAM role is invalid."
  resolution: "Check instance type availability. Verify IAM role exists and has correct trust policy for sagemaker.amazonaws.com."

- symptoms: "Braket SDK import fails in notebook"
  diagnosis: "SDK is not installed or is an incompatible version."
  resolution: "Run pip install --upgrade amazon-braket-sdk in the notebook terminal."

## Output Format

```yaml
root_cause: "notebook_instance — <specific_cause>"
evidence:
  - type: instance_status
    content: "<notebook instance status and error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix instance configuration or IAM role"
  long_term: "Use lifecycle configurations for consistent setup"
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
