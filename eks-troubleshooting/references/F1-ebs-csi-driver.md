---
title: "F1 — EBS CSI Driver Issues"
description: "Diagnose and resolve Amazon EBS CSI driver problems on EKS"
status: active
severity: HIGH
triggers:
  - "EBS CSI"
  - "ebs-csi"
  - "volume attach"
  - "volume provision"
  - "gp3"
  - "gp2"
owner: devops-agent
objective: "Identify and fix EBS CSI driver issues to restore persistent volume functionality"
context: "The EBS CSI driver is required for dynamic EBS volume provisioning on EKS 1.23+. It runs as a Deployment (controller) and DaemonSet (node) in kube-system. The driver needs IAM permissions to create, attach, and manage EBS volumes. It's available as an EKS managed add-on."
---

## Phase 1 — Triage

MUST:
- Check if the EBS CSI driver add-on is installed: `aws eks describe-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver`
- Check CSI driver pods: `kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver`
- Check CSI controller logs: `kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin --tail=50`
- Check CSI node logs: `kubectl logs -n kube-system -l app=ebs-csi-node -c ebs-plugin --tail=50`
- Verify the StorageClass exists: `kubectl get storageclass`

SHOULD:
- Check the CSI driver's service account and IRSA: `kubectl get sa ebs-csi-controller-sa -n kube-system -o yaml`
- Verify IAM role permissions: needs `ec2:CreateVolume`, `ec2:AttachVolume`, `ec2:DetachVolume`, `ec2:DeleteVolume`, `ec2:CreateSnapshot`, `ec2:DescribeVolumes`, etc.
- Check CSIDriver resource: `kubectl get csidriver ebs.csi.aws.com -o yaml`
- Verify the add-on version is compatible with the cluster version

MAY:
- Check VolumeAttachment resources: `kubectl get volumeattachment`
- Check for stuck volume attachments on terminated nodes
- Review CSI driver configuration values

## Phase 2 — Remediate

MUST:
- If add-on not installed: install it: `aws eks create-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver --service-account-role-arn <role-arn>`
- If IAM permissions missing: create/update the IAM role with the EBS CSI driver policy
- If controller pods are failing: check logs for specific errors, restart the deployment
- If StorageClass missing: create one referencing `ebs.csi.aws.com` provisioner

SHOULD:
- Use the EKS managed add-on instead of self-managed Helm installation
- Set up IRSA for the CSI driver (don't rely on node role permissions)
- Create a default StorageClass for the cluster

MAY:
- Configure volume encryption by default in the StorageClass
- Set up volume snapshot capability with the CSI snapshotter

## Common Issues

- symptoms: "PVC stuck in Pending, events show 'failed to provision volume'"
  diagnosis: "EBS CSI driver not installed or IAM permissions insufficient."
  resolution: "Install the EBS CSI driver add-on with proper IAM role."

- symptoms: "Volume created but pod can't mount it — 'Multi-Attach error'"
  diagnosis: "EBS volume is still attached to a terminated/different node."
  resolution: "Force detach the volume: `aws ec2 detach-volume --volume-id <vol-id> --force`. Wait for detach, then delete the VolumeAttachment."

- symptoms: "CSI controller logs show 'AccessDenied' errors"
  diagnosis: "CSI driver's IAM role lacks required EC2/EBS permissions."
  resolution: "Update the IAM role with the Amazon_EBS_CSI_Driver managed policy or equivalent."

## Output Format

```yaml
root_cause: "EBS CSI driver — <specific_cause>"
evidence:
  - type: addon_status
    content: "<EBS CSI driver add-on status>"
  - type: csi_logs
    content: "<relevant CSI controller/node logs>"
severity: HIGH
mitigation:
  immediate: "Fix CSI driver installation or IAM permissions"
  long_term: "Use EKS managed add-on with IRSA, implement volume monitoring"
```

## Safety Ratings
- GREEN: read-only (`aws eks describe-addon`, `kubectl get pods`, `kubectl logs`, `kubectl get storageclass`, `kubectl get pv`, `kubectl get volumeattachment`)
- YELLOW: state-changing recoverable (`aws eks create-addon`, `aws eks update-addon`, `kubectl apply storageclass`, update CSI driver IAM role, `kubectl delete pod` to restart CSI pods)
- RED: destructive/irreversible (`aws ec2 detach-volume --force`, `kubectl delete pv`, deleting EBS CSI driver add-on without `--preserve`)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves force-detaching EBS volumes from production nodes"
- "Remediation requires modifying CSI driver IAM permissions in production"

## Rollback
- Pre-change: "Back up aws-auth ConfigMap before editing"
- Pre-change: "Save current StorageClass configuration before modification"
- Pre-change: "Save current RBAC bindings before modification"
- Verification: "Verify PVC binding and pod volume mounts after change"
- Revert: "Restore StorageClass from backup if provisioning breaks"

## Data Sensitivity
- HIGH: "aws-auth ConfigMap reveals all IAM-to-RBAC mappings"
- HIGH: "Service account annotations reveal IAM role ARNs (IRSA)"
- MEDIUM: "Node group configuration reveals instance types and scaling"
- MEDIUM: "StorageClass parameters reveal encryption settings and volume configuration"
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
