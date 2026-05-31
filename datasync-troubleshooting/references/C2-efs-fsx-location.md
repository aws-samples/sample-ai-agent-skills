---
title: "C2 — EFS/FSx Location Issues"
description: "Diagnose EFS and FSx location configuration problems in DataSync"
status: active
severity: HIGH
triggers:
  - "EFS location error"
  - "FSx location error"
  - "cannot mount EFS"
  - "FSx access denied"
owner: devops-agent
objective: "Identify and resolve EFS or FSx location configuration issues in DataSync"
context: "EFS and FSx locations require proper VPC configuration, security groups, and subnet access. DataSync accesses these filesystems through ENIs in specified subnets. Security groups must allow NFS (port 2049) for EFS or SMB (port 445) for FSx for Windows."
---

## Phase 1 — Triage

MUST:
- Check location configuration: `aws datasync describe-location-efs --location-arn <location-arn>` or `aws datasync describe-location-fsx-windows --location-arn <location-arn>`
- Verify filesystem exists and is available: `aws efs describe-file-systems --file-system-id <fs-id>` or `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check security groups allow DataSync ENI access: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify subnet configuration and VPC connectivity

SHOULD:
- Check EFS mount targets exist in the specified subnet: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Verify FSx endpoint IPs are reachable from DataSync subnet
- Check EFS access points if configured: `aws efs describe-access-points --file-system-id <fs-id>`

MAY:
- Check VPC Flow Logs for blocked traffic between DataSync ENI and filesystem
- Verify EFS file system policy allows DataSync access

## Phase 2 — Remediate

MUST:
- Ensure security groups allow inbound NFS (2049) for EFS or SMB (445) for FSx from DataSync ENI security group
- Verify DataSync subnet has route to filesystem subnet
- For EFS: ensure mount target exists in the DataSync subnet's AZ

SHOULD:
- Create EFS location: `aws datasync create-location-efs --efs-filesystem-arn <efs-arn> --ec2-config SubnetArn=<subnet-arn>,SecurityGroupArns=<sg-arns> --subdirectory /path/`
- Create FSx location: `aws datasync create-location-fsx-windows --fsx-filesystem-arn <fsx-arn> --security-group-arns <sg-arns> --subdirectory /share/ --user <user> --password <password>`

MAY:
- Use EFS access points for fine-grained access control
- Configure FSx for Windows with Active Directory credentials

## Common Issues

- symptoms: "Mount failed for EFS location"
  diagnosis: "Security group blocking port 2049 or no mount target in subnet's AZ."
  resolution: "Add inbound rule for port 2049. Create mount target in the correct AZ."

- symptoms: "FSx location creation fails with access denied"
  diagnosis: "Invalid SMB credentials or security group blocking port 445."
  resolution: "Verify SMB user/password. Allow port 445 inbound on FSx security group."

- symptoms: "Timeout connecting to filesystem"
  diagnosis: "No network route between DataSync subnet and filesystem subnet."
  resolution: "Verify subnets are in the same VPC or have proper peering/transit gateway routes."

## Output Format

```yaml
root_cause: "efs_fsx_location_issue — <specific_cause>"
evidence:
  - type: location_config
    content: "<EFS/FSx location configuration>"
  - type: filesystem_status
    content: "<filesystem availability and mount targets>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix security groups and network configuration"
  long_term: "Document filesystem access requirements for DataSync"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — EFS/FSx filesystems may contain sensitive application data |
| Blast Radius | MEDIUM — Security group changes affect all services using the filesystem |
| Rollback Complexity | LOW — Revert security group changes; recreate location |
| Compliance Scope | HIGH — Filesystem access must comply with data governance policies |

## Escalation Conditions

- Escalate to security if security group changes expose filesystem to unintended access
- Escalate to application owners if DataSync ENIs affect filesystem performance
- Escalate to network team if VPC routing issues prevent filesystem access

## Data Sensitivity

- **Filesystem contents** may be highly sensitive — verify access controls
- **Security group rules** opened for DataSync also affect other services
- **SMB credentials** for FSx for Windows must be stored securely

## PROHIBITED Actions

- Do NOT open filesystem security groups to 0.0.0.0/0 to fix connectivity
- Do NOT store SMB credentials in plain text
- Do NOT modify filesystem security groups without understanding the impact on other services
- Do NOT create DataSync ENIs in public subnets

## Phase 3 — Rollback

- Revert security group changes if they were modified during troubleshooting
- Delete the misconfigured location and recreate with correct settings
- Verify filesystem access is restored for all dependent services

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
