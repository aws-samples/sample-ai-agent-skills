---
title: "A3 — AMI Launch Failures"
description: "Diagnose EC2 launch failures caused by AMI issues (permissions, region, architecture, encryption)"
status: active
severity: MEDIUM
triggers:
  - "InvalidAMIID"
  - "AMI.*not found"
  - "not authorized.*image"
  - "does not exist in.*region"
  - "architecture.*not supported"
owner: devops-agent
objective: "Identify the AMI issue and launch the instance with a valid AMI"
context: "AMIs are region-specific, architecture-specific, and permission-controlled. Common failures: AMI not shared to account, AMI in wrong region, ARM AMI on x86 instance type, encrypted AMI without KMS access."
---

## Phase 1 — Triage

MUST:
- Verify the AMI ID exists in the target region: `aws ec2 describe-images --image-ids <ami-id>`
- Check AMI architecture (x86_64 vs arm64) matches instance type architecture
- Check AMI permissions if using a shared/community AMI
- Check if AMI is encrypted and if the launch role has KMS decrypt permissions

SHOULD:
- Verify the AMI is not deprecated or deregistered
- Check if the AMI's root device type (ebs vs instance-store) is compatible with the instance type
- Verify ENA support flag matches instance type requirements (Nitro instances require ENA)

MAY:
- Check if the AMI was recently deregistered by the owner
- Verify boot mode (UEFI vs legacy BIOS) compatibility with instance type

## Common Issues

- symptoms: "InvalidAMIID.NotFound"
  diagnosis: "AMI does not exist in the target region. AMIs are region-specific."
  resolution: "Copy the AMI to the target region: `aws ec2 copy-image --source-region <src> --source-image-id <ami> --region <dst>`"

- symptoms: "Launch fails with architecture mismatch"
  diagnosis: "ARM AMI (arm64) on x86 instance type or vice versa."
  resolution: "Use matching instance type: arm64 AMI → Graviton instances (*.g suffix), x86_64 AMI → Intel/AMD instances."

- symptoms: "Launch fails with 'not authorized' on encrypted AMI"
  diagnosis: "Launch role lacks KMS permissions for the AMI's encryption key."
  resolution: "Grant kms:Decrypt and kms:CreateGrant on the CMK to the launch role."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-images to verify AMI: GREEN — read-only API call"
  - "copy-image to copy AMI to target region: GREEN — creates new resource, no modification"
  - "Grant KMS permissions for encrypted AMI: YELLOW — IAM/KMS policy change, recoverable"
  - "Deregister old AMI and re-register: RED — original AMI ID lost permanently"
```

## Escalation Conditions
- AMI is owned by another account and permissions cannot be verified
- Encrypted AMI requires KMS key policy changes in a production account
- AMI is the only copy and shows signs of corruption
- Launch failure affects a production Auto Scaling group with no healthy instances
- AMI architecture mismatch requires application recompilation

## Data Sensitivity
- HIGH: describe-images (reveals AMI ownership, encryption keys, account IDs of shared AMIs)
- MEDIUM: describe-instance-attribute (may reveal user data scripts with embedded secrets)
- LOW: describe-instance-type-offerings (public instance type availability data)

## Prohibited Actions
- NEVER suggest deregistering an AMI without confirming backups or other copies exist
- NEVER suggest making a private AMI public to resolve permission issues
- NEVER suggest disabling AMI encryption to work around KMS permission errors
- NEVER suggest using community/marketplace AMIs without security review

## Phase 3 — Rollback
- If AMI was copied to new region: deregister the copy if it's not needed (and associated snapshots)
- If KMS permissions were modified: revert IAM/KMS policy to previous version
- If launch template was updated with new AMI: create new template version pointing to original AMI
- If AMI sharing permissions were changed: revoke sharing with `modify-image-attribute --launch-permission`

## Output Format

```yaml
root_cause: "<ami_not_found|architecture_mismatch|permission_denied|encryption> — <detail>"
evidence:
  - type: api_error
    content: "<error from RunInstances or describe-images>"
severity: MEDIUM
mitigation:
  immediate: "Use correct AMI for region/architecture/permissions"
  long_term: "Automate AMI validation in launch templates"
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
