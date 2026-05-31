---
title: "D1 â€” EFS CSI Driver Issues"
description: "Diagnose EFS CSI driver problems in EKS"
status: active
severity: HIGH
triggers:
  - "EFS CSI"
  - "efs-csi-driver"
  - "CSI driver EFS"
  - "PV EFS"
owner: devops-agent
objective: "Identify and fix EFS CSI driver installation and configuration issues"
context: "The EFS CSI driver enables EKS pods to use EFS file systems via PersistentVolumes. It requires mount targets in worker node subnets, the driver DaemonSet running on nodes, and correct StorageClass/PV/PVC configuration."
---

## Phase 1 â€” Triage

MUST:
- Check CSI driver is installed: `kubectl get daemonset efs-csi-node -n kube-system`
- Check CSI driver pods are running: `kubectl get pods -n kube-system -l app=efs-csi-node`
- Verify mount targets exist in node subnets: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Check PV and PVC status: `kubectl get pv,pvc`

SHOULD:
- Check CSI driver logs: `kubectl logs -n kube-system -l app=efs-csi-node -c efs-plugin`
- Verify the CSI driver version is compatible
- Check if the driver has the correct IAM permissions (via IRSA or node role)

MAY:
- Check EFS CSI controller pods: `kubectl get pods -n kube-system -l app=efs-csi-controller`
- Verify StorageClass configuration

## Phase 2 â€” Remediate

MUST:
- Install or update the EFS CSI driver if missing
- Create mount targets in all AZs where worker nodes run
- Fix PV/PVC configuration to reference correct file system ID

SHOULD:
- Use IRSA for CSI driver IAM permissions
- Ensure security groups allow NFS traffic from node security group

MAY:
- Enable CSI driver logging for debugging

## Common Issues

- symptoms: "PVC stuck in Pending state"
  diagnosis: "No mount target in the worker node's AZ or CSI driver not installed."
  resolution: "Create mount targets in all node AZs. Verify CSI driver DaemonSet is running."

- symptoms: "Pod fails to start with mount error"
  diagnosis: "CSI driver pod is not running on the node or security group blocks NFS."
  resolution: "Check CSI driver DaemonSet. Verify security group allows port 2049."

- symptoms: "CSI driver pods in CrashLoopBackOff"
  diagnosis: "Driver version incompatibility or missing IAM permissions."
  resolution: "Update CSI driver. Check IAM role has EFS permissions."

## Output Format

```yaml
root_cause: "efs_csi_driver_issue â€” <specific_cause>"
evidence:
  - type: csi_driver_status
    content: "<DaemonSet and pod status>"
  - type: mount_targets
    content: "<mount target AZ coverage>"
severity: HIGH
mitigation:
  immediate: "Fix CSI driver installation or mount target coverage"
  long_term: "Automate mount target creation with infrastructure as code"
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
