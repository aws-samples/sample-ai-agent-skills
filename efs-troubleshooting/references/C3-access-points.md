---
title: "C3 â€” EFS Access Point Issues"
description: "Diagnose access point configuration and POSIX permission problems"
status: active
severity: MEDIUM
triggers:
  - "access point"
  - "POSIX permissions"
  - "uid gid"
  - "permission denied EFS"
owner: devops-agent
objective: "Identify and fix access point configuration issues"
context: "Access points enforce POSIX user/group identity and can set a root directory. They override NFS client credentials. Useful for multi-tenant and container workloads."
---

## Phase 1 â€” Triage

MUST:
- Check access point configuration: `aws efs describe-access-points --file-system-id <fs-id>`
- Verify POSIX user settings (UID, GID, secondary GIDs)
- Check root directory path and creation info
- Verify the mount command references the access point

SHOULD:
- Check if the root directory exists and has correct permissions
- Verify the access point is in Available state
- Check if IAM authorization is required for the access point

MAY:
- List files in the root directory to verify access
- Check if multiple access points have conflicting configurations

## Phase 2 â€” Remediate

MUST:
- Fix POSIX user/group settings to match application requirements
- Mount with access point: `sudo mount -t efs -o accesspoint=<ap-id> <fs-id>:/ /mnt/efs`
- Create root directory with correct permissions if it doesn't exist

SHOULD:
- Use access points with IAM for secure multi-tenant access
- Document access point configurations per application

MAY:
- Create separate access points per application or team

## Common Issues

- symptoms: "Permission denied when writing through access point"
  diagnosis: "The access point's POSIX UID/GID doesn't have write permission on the directory."
  resolution: "Adjust the root directory permissions or change the access point's POSIX identity."

- symptoms: "Access point mount shows empty directory"
  diagnosis: "Root directory path doesn't exist or is different from expected."
  resolution: "Check the access point's root directory path. Create it if needed."

- symptoms: "Files created through access point have unexpected ownership"
  diagnosis: "Access point enforces its configured UID/GID regardless of client identity."
  resolution: "This is expected behavior. Adjust the access point's POSIX settings if needed."

## Output Format

```yaml
root_cause: "access_point_issue â€” <specific_cause>"
evidence:
  - type: access_point_config
    content: "<access point POSIX and root directory settings>"
severity: MEDIUM
mitigation:
  immediate: "Fix access point configuration or directory permissions"
  long_term: "Document and standardize access point configurations"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL — File system contains application data and shared state |
| Recovery Complexity | HIGH — Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH — Mount or performance issues affect all connected instances |

## Escalation Conditions

- File system unavailable or mount failures affecting multiple production instances
- Throughput degradation causing application timeouts across the fleet
- Burst credits depleted with no provisioned throughput configured
- Security group changes blocking NFS access for running workloads
- Replication lag exceeding RPO thresholds for disaster recovery
- IAM authorization failures preventing application access to file system

## Data Sensitivity

- File system contents may include application data, configuration files, and shared secrets
- Access point configurations reveal application directory structure and POSIX permissions
- Mount target IPs and DNS names are internal infrastructure identifiers
- Encryption keys (KMS) protect data at rest — key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 — Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
