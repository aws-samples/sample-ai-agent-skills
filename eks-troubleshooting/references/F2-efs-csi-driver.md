---
title: "F2 — EFS CSI Driver Issues"
description: "Diagnose and resolve Amazon EFS CSI driver problems on EKS"
status: active
severity: HIGH
triggers:
  - "EFS CSI"
  - "efs-csi"
  - "EFS mount"
  - "shared storage"
  - "ReadWriteMany"
owner: devops-agent
objective: "Identify and fix EFS CSI driver issues to restore shared persistent volume functionality"
context: "The EFS CSI driver enables EFS file systems as persistent volumes in EKS. Unlike EBS, EFS supports ReadWriteMany (RWX) access mode — multiple pods across multiple nodes can mount the same volume. The driver needs network connectivity to EFS mount targets and proper security group configuration."
---

## Phase 1 — Triage

MUST:
- Check if the EFS CSI driver add-on is installed: `aws eks describe-addon --cluster-name <cluster> --addon-name aws-efs-csi-driver`
- Check CSI driver pods: `kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver`
- Check CSI controller and node logs: `kubectl logs -n kube-system -l app=efs-csi-controller -c efs-plugin --tail=50`
- Verify the EFS file system exists: `aws efs describe-file-systems --file-system-id <fs-id>`
- Check mount targets in the cluster's subnets: `aws efs describe-mount-targets --file-system-id <fs-id>`

SHOULD:
- Verify security groups allow NFS traffic (port 2049) from node security group to EFS mount target security group
- Check the PersistentVolume spec references the correct EFS file system ID
- Verify the StorageClass uses `efs.csi.aws.com` provisioner
- Check if the EFS file system and cluster are in the same VPC

MAY:
- Check EFS access points if using dynamic provisioning: `aws efs describe-access-points --file-system-id <fs-id>`
- Verify IAM permissions for the CSI driver (needed for dynamic provisioning)
- Check EFS throughput mode and performance metrics

## Phase 2 — Remediate

MUST:
- If add-on not installed: install it: `aws eks create-addon --cluster-name <cluster> --addon-name aws-efs-csi-driver`
- If mount target missing: create mount targets in the cluster's subnets
- If security group blocking: add inbound NFS (2049) rule to the EFS mount target security group from the node security group
- If PV spec wrong: fix the EFS file system ID and mount options

SHOULD:
- Create mount targets in every AZ where nodes run
- Use EFS access points for dynamic provisioning (one access point per PVC)
- Set up IRSA for the EFS CSI driver

MAY:
- Configure EFS encryption in transit via mount options
- Use EFS Intelligent-Tiering for cost optimization

## Common Issues

- symptoms: "Pod stuck in ContainerCreating, events show 'mount failed: timed out'"
  diagnosis: "Node cannot reach the EFS mount target — security group or no mount target in the AZ."
  resolution: "Check security groups (port 2049). Ensure mount targets exist in the node's AZ."

- symptoms: "PVC Pending with 'failed to provision volume with StorageClass efs-sc'"
  diagnosis: "EFS CSI driver not installed or dynamic provisioning not configured."
  resolution: "Install the EFS CSI driver. For dynamic provisioning, configure StorageClass with EFS file system ID."

- symptoms: "Mount succeeds but writes fail with 'Permission denied'"
  diagnosis: "EFS access point UID/GID doesn't match the container's user."
  resolution: "Configure the access point with the correct POSIX user (UID/GID) or use securityContext in the pod."

## Output Format

```yaml
root_cause: "EFS CSI driver — <specific_cause>"
evidence:
  - type: addon_status
    content: "<EFS CSI driver add-on status>"
  - type: mount_targets
    content: "<EFS mount target configuration>"
severity: HIGH
mitigation:
  immediate: "Fix CSI driver, mount targets, or security group configuration"
  long_term: "Automate EFS mount target creation in IaC, implement EFS monitoring"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-addon`, `kubectl get pods`, `kubectl logs`, `aws efs describe-file-systems`, `aws efs describe-mount-targets`, `aws efs describe-access-points`)
- YELLOW: state-changing recoverable (`aws eks create-addon`, `aws eks update-addon`, `aws efs create-mount-target`, update security group rules for NFS port 2049, `kubectl apply` PV/StorageClass)
- RED: destructive/irreversible (`aws efs delete-file-system`, `aws efs delete-mount-target` in production, deleting EFS CSI driver add-on without `--preserve`)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying security groups for EFS mount targets in production"
- "Remediation requires creating or deleting EFS mount targets in production subnets"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current security group rules before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify EFS mount and pod volume access after change"
- Revert: "Restore security group rules from backup if NFS connectivity breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "EFS access point configuration reveals POSIX user/group mappings"
- LOW: "Pod status and events"

## Prohibited Actions
- NEVER suggest adding `system:masters` group for troubleshooting access
- NEVER suggest disabling Pod Security Standards/Admission
- NEVER suggest running containers as root to fix permission issues
- NEVER suggest `kubectl delete namespace` in production without confirmation
- NEVER suggest modifying kube-system resources without backup

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
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
