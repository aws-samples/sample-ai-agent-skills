---
title: "Z1 — General EFS Troubleshooting (Catch-All)"
description: "Fallback SOP for EFS issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown EFS issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check file system status: `aws efs describe-file-systems --file-system-id <fs-id>`
- Check mount targets: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Check security groups: `aws efs describe-mount-target-security-groups --mount-target-id <mt-id>`
- Check lifecycle configuration: `aws efs describe-lifecycle-configuration --file-system-id <fs-id>`
- Check file system policy: `aws efs describe-file-system-policy --file-system-id <fs-id>`
- Check CloudWatch EFS metrics for anomalies

SHOULD:
- Check access points: `aws efs describe-access-points --file-system-id <fs-id>`
- Check replication: `aws efs describe-replication-configurations --file-system-id <fs-id>`
- Review CloudTrail for recent EFS API calls

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Mount failures → Mount Issues (A1-A3)
- Throughput/latency → Performance (B1-B3)
- Permission/access → Security (C1-C3)
- EKS/container → EKS Integration (D1-D2)
- Storage class/replication → Lifecycle & Replication (E1-E2)
- Encryption → Encryption (F1-F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all file system configurations systematically
- Review CloudTrail for recent configuration changes
- Check CloudWatch metrics for all EFS dimensions
- Verify network connectivity from affected clients

SHOULD:
- Check AWS Health Dashboard for EFS service events
- Compare with a known-good file system configuration

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<mount|performance|security|eks|lifecycle|encryption|unknown>"
investigation_path: "file system config → mount targets → security groups → metrics → <domain_classification>"
evidence:
  - type: file_system_config
    content: "<configuration summary>"
  - type: cloudwatch_metrics
    content: "<relevant metrics>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH � EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL � File system contains application data and shared state |
| Recovery Complexity | HIGH � Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH � Mount or performance issues affect all connected instances |

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
- Encryption keys (KMS) protect data at rest � key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 � Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
