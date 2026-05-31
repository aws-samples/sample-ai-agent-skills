---
title: "G1 — Backup and Restore"
description: "Diagnose EKS Anywhere cluster backup and restore issues"
status: active
severity: CRITICAL
triggers: ["backup failed", "restore failed", "etcd backup error", "Velero issue", "cluster recovery", "disaster recovery"]
owner: devops-agent
objective: "Resolve backup and restore issues including etcd snapshots, Velero backups, and cluster recovery"
context: "EKS Anywhere cluster state is stored in etcd. Backups can be taken via etcd snapshots or Velero. Restoring requires careful handling to avoid split-brain or data corruption."
---

## Phase 1 — Triage

MUST:
- Check etcd cluster health: `kubectl exec -n kube-system etcd-<node> -- etcdctl endpoint health --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key`
- Verify backup storage accessibility (S3, NFS, local storage)
- Check Velero status (if installed): `kubectl get pods -n velero --kubeconfig <workload-kubeconfig>`

SHOULD:
- List existing backups: `velero backup get` or check etcd snapshot files
- Review backup schedule: `velero schedule get`
- Check backup storage location status: `velero backup-location get`

MAY:
- Verify etcd snapshot integrity: `etcdctl snapshot status <snapshot-file>`
- Check backup encryption configuration

## Phase 2 — Remediate

MUST:
- Fix backup storage connectivity (credentials, network, permissions)
- Resolve etcd snapshot failures (disk space, permissions, etcd health)
- Ensure backup schedule is active and producing valid backups

SHOULD:
- Test restore procedure in a non-production environment
- Verify backup completeness (all namespaces, PVs, CRDs included)

MAY:
- Configure backup retention policies
- Set up backup monitoring and alerting

## Safety Ratings

```
safety_ratings:
  - "etcdctl endpoint health: GREEN — read-only health check"
  - "velero backup get: GREEN — read-only backup listing"
  - "etcdctl snapshot save: YELLOW — creates snapshot, brief I/O impact on etcd"
  - "etcdctl snapshot restore: RED — replaces etcd data, requires cluster downtime"
```

## Escalation Conditions

- etcd cluster is unhealthy and no valid backup exists
- Restore from backup fails with data corruption errors
- Backup storage is inaccessible and backups cannot be created or retrieved
- Management cluster needs recovery and workload clusters are affected

## Data Sensitivity

- `etcdctl endpoint health`: LOW — returns health status
- `etcd snapshots`: CRITICAL — contain all cluster state including secrets
- `velero backup get`: LOW — returns backup metadata
- `backup storage credentials`: HIGH — provide access to all backup data

## Prohibited Actions

- NEVER restore an etcd snapshot on a running cluster without stopping the API server first
- NEVER delete etcd snapshots without verifying newer valid backups exist
- NEVER store etcd snapshots in unencrypted public storage
- NEVER perform restore operations on a production cluster without a tested procedure

## Phase 3 — Rollback

1. If a restore was performed incorrectly, restore from the pre-restore backup (if taken)
2. If etcd was corrupted during restore, use the original snapshot and retry with correct procedure
3. If Velero configuration was changed, restore from previous configuration
4. Verify cluster state after restore: `kubectl get nodes`, `kubectl get pods -A`
5. Validate workload health and data integrity after restore

## Common Issues

- symptoms: "etcd snapshot save fails — no space left on device"
  diagnosis: "Insufficient disk space on the control plane node for etcd snapshots."
  resolution: "Free disk space or configure snapshot storage to an external volume."

## Output Format

```yaml
root_cause: "backup_restore — <specific_cause>"
evidence:
  - type: backup_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix backup/restore issue"
  long_term: "Implement automated backup testing and monitoring"
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
