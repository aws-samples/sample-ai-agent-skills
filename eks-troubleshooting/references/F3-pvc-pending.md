---
title: "F3 — PVC Pending"
description: "Diagnose and resolve PersistentVolumeClaim stuck in Pending state"
status: active
severity: HIGH
triggers:
  - "PVC pending"
  - "PersistentVolumeClaim"
  - "waiting for a volume"
  - "no persistent volumes available"
owner: devops-agent
objective: "Identify why the PVC cannot be bound and resolve the storage provisioning issue"
context: "PVCs go Pending when no matching PersistentVolume exists and dynamic provisioning fails. Common causes: missing CSI driver, wrong StorageClass, insufficient IAM permissions, AZ mismatch between pod and volume, or storage quota exceeded."
---

## Phase 1 — Triage

MUST:
- Check PVC status and events: `kubectl describe pvc <pvc-name> -n <namespace>`
- Check the StorageClass referenced by the PVC: `kubectl get storageclass <sc-name> -o yaml`
- Verify the CSI driver for the StorageClass is installed: `kubectl get csidriver`
- Check if there are matching PVs (for static provisioning): `kubectl get pv`
- Check the provisioner in the StorageClass (ebs.csi.aws.com, efs.csi.aws.com, or legacy kubernetes.io/aws-ebs)

SHOULD:
- Check CSI driver pod logs for provisioning errors
- Verify the StorageClass parameters (type, fsType, encrypted, etc.)
- Check if the PVC's access mode is supported by the storage type (EBS = RWO only, EFS = RWX)
- Check for volume binding mode: `WaitForFirstConsumer` vs `Immediate`

MAY:
- Check storage quotas in the namespace: `kubectl get resourcequota -n <namespace>`
- Check AWS service quotas for EBS volumes
- Verify the AZ of the requesting pod matches available storage

## Phase 2 — Remediate

MUST:
- If CSI driver missing: install the appropriate CSI driver (EBS or EFS)
- If StorageClass wrong: create the correct StorageClass or update the PVC
- If legacy provisioner: migrate from `kubernetes.io/aws-ebs` to `ebs.csi.aws.com`
- If AZ mismatch: use `WaitForFirstConsumer` volume binding mode to provision in the pod's AZ

SHOULD:
- Set a default StorageClass: `kubectl patch storageclass <sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
- Use `WaitForFirstConsumer` binding mode to avoid AZ mismatch issues
- Ensure CSI driver has proper IAM permissions

MAY:
- Pre-provision PVs for specific workloads (static provisioning)
- Set up storage monitoring and alerting

## Common Issues

- symptoms: "PVC Pending with 'no persistent volumes available for this claim'"
  diagnosis: "No matching PV exists and dynamic provisioning is not configured or failing."
  resolution: "Check StorageClass and CSI driver. Ensure dynamic provisioning is working."

- symptoms: "PVC Pending with 'waiting for first consumer to be created before binding'"
  diagnosis: "StorageClass uses WaitForFirstConsumer — PVC waits until a pod references it."
  resolution: "This is normal behavior. Create a pod that references the PVC."

- symptoms: "PVC Pending, CSI driver logs show 'could not create volume in zone us-east-1a'"
  diagnosis: "Volume cannot be created in the required AZ — capacity or quota issue."
  resolution: "Check EBS quotas. Try a different AZ or instance type."

- symptoms: "PVC Pending with StorageClass using kubernetes.io/aws-ebs provisioner"
  diagnosis: "Legacy in-tree provisioner is deprecated on EKS 1.23+."
  resolution: "Install the EBS CSI driver and create a StorageClass using ebs.csi.aws.com."

## Output Format

```yaml
root_cause: "PVC Pending — <specific_cause>"
evidence:
  - type: pvc_events
    content: "<PVC describe events>"
  - type: storage_class
    content: "<StorageClass configuration>"
severity: HIGH
mitigation:
  immediate: "Fix CSI driver, StorageClass, or provisioning configuration"
  long_term: "Use WaitForFirstConsumer binding mode, implement storage monitoring"
```

## Safety Ratings
- GREEN: read-only (`kubectl describe pvc`, `kubectl get storageclass`, `kubectl get csidriver`, `kubectl get pv`, `kubectl get resourcequota`)
- YELLOW: state-changing recoverable (`kubectl apply storageclass`, `kubectl patch storageclass` to set default, `kubectl apply` PV for static provisioning, install CSI driver add-on)
- RED: destructive/irreversible (`kubectl delete pv` with Retain policy data, `kubectl delete pvc` with bound volumes in production)

## Escalation Conditions
- "Remediation requires editing aws-auth ConfigMap in production"
- "Fix involves modifying RBAC ClusterRoleBindings"
- "Change affects node group scaling in production"
- "Fix involves modifying StorageClass defaults in production cluster"
- "Remediation requires migrating from legacy in-tree provisioner to CSI driver"

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
- MEDIUM: "StorageClass parameters reveal encryption and volume type settings"
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
