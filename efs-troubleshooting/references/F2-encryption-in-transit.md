---
title: "F2 â€” EFS Encryption in Transit Issues"
description: "Diagnose TLS encryption in transit configuration for EFS"
status: active
severity: MEDIUM
triggers:
  - "encryption in transit"
  - "TLS EFS"
  - "mount helper TLS"
  - "stunnel"
owner: devops-agent
objective: "Identify and fix encryption in transit configuration issues"
context: "EFS encryption in transit uses TLS via the amazon-efs-utils mount helper. It requires the -o tls mount option. The mount helper uses stunnel to create a TLS tunnel. Standard NFS mount does not encrypt traffic."
---

## Phase 1 â€” Triage

MUST:
- Check if amazon-efs-utils is installed on the client
- Verify the mount command uses -o tls option
- Check if stunnel is installed (required by the mount helper)
- Check mount helper logs: `/var/log/amazon/efs/mount.log`

SHOULD:
- Verify the file system policy requires encryption in transit
- Check if stunnel process is running after mount
- Verify the EFS mount helper version is current

MAY:
- Check if the file system policy denies unencrypted connections
- Verify certificate validation is working

## Phase 2 â€” Remediate

MUST:
- Install amazon-efs-utils if missing
- Mount with TLS: `sudo mount -t efs -o tls <fs-id>:/ /mnt/efs`
- Install stunnel if missing

SHOULD:
- Update /etc/fstab to include tls option for persistent mounts
- Enforce encryption in transit via file system policy

MAY:
- Configure custom stunnel settings in /etc/amazon/efs/efs-utils.conf

## Common Issues

- symptoms: "Mount with -o tls fails"
  diagnosis: "amazon-efs-utils or stunnel not installed."
  resolution: "Install amazon-efs-utils (includes stunnel dependency)."

- symptoms: "Mount succeeds but traffic is not encrypted"
  diagnosis: "Mount command doesn't include -o tls option."
  resolution: "Remount with -o tls. Update /etc/fstab."

- symptoms: "File system policy denies connection"
  diagnosis: "Policy requires aws:SecureTransport but client is not using TLS."
  resolution: "Mount with -o tls to enable encryption in transit."

## Output Format

```yaml
root_cause: "encryption_in_transit_issue â€” <specific_cause>"
evidence:
  - type: mount_config
    content: "<mount command and options>"
  - type: mount_helper_logs
    content: "<relevant log entries>"
severity: MEDIUM
mitigation:
  immediate: "Install mount helper and remount with TLS"
  long_term: "Enforce encryption in transit via file system policy"
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
