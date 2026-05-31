---
title: "D2 â€” EFS PVC Issues"
description: "Diagnose PersistentVolumeClaim issues with EFS in EKS"
status: active
severity: HIGH
triggers:
  - "PVC pending"
  - "PersistentVolumeClaim"
  - "volume mount failed"
  - "EFS PVC"
owner: devops-agent
objective: "Identify and fix PVC binding and mounting issues with EFS"
context: "EFS PVCs in EKS require a PersistentVolume with the correct CSI driver reference, file system ID, and optionally an access point. Static provisioning is most common for EFS."
---

## Phase 1 â€” Triage

MUST:
- Check PVC status: `kubectl describe pvc <pvc-name>`
- Check PV configuration: `kubectl get pv -o yaml`
- Verify the PV references the correct EFS file system ID
- Check events on the PVC: `kubectl get events --field-selector involvedObject.name=<pvc-name>`

SHOULD:
- Verify the StorageClass exists and is correct
- Check if the PV and PVC have matching accessModes and storageClassName
- Verify the CSI driver is specified in the PV

MAY:
- Check pod events for mount failures
- Verify access point ID in the PV if using access points

## Phase 2 â€” Remediate

MUST:
- Fix PV to reference correct file system ID and CSI driver
- Ensure PV and PVC have matching accessModes (ReadWriteMany for EFS)
- Fix StorageClass reference

SHOULD:
- Use access points for per-pod isolation
- Set volumeHandle to a unique value per PV

MAY:
- Implement dynamic provisioning with EFS CSI driver

## Common Issues

- symptoms: "PVC stuck in Pending â€” no matching PV"
  diagnosis: "PV doesn't exist or accessModes/storageClassName don't match."
  resolution: "Create a PV with matching accessModes (ReadWriteMany) and storageClassName."

- symptoms: "Pod stuck in ContainerCreating with mount error"
  diagnosis: "PV has wrong file system ID or mount target is missing."
  resolution: "Verify file system ID in PV. Ensure mount targets exist in node AZs."

- symptoms: "Multiple pods can't share the same PVC"
  diagnosis: "PV accessMode is not ReadWriteMany."
  resolution: "Set accessModes to ReadWriteMany for EFS PVs."

## Output Format

```yaml
root_cause: "pvc_issue â€” <specific_cause>"
evidence:
  - type: pvc_status
    content: "<PVC describe output>"
  - type: pv_config
    content: "<PV configuration>"
severity: HIGH
mitigation:
  immediate: "Fix PV/PVC configuration"
  long_term: "Standardize EFS PV/PVC templates"
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
