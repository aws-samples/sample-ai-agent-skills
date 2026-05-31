---
title: "Z1 — General FSx Troubleshooting"
description: "Catch-all runbook for FSx issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "fsx problem"
  - "fsx issue"
  - "fsx not working"
  - "fsx error"
  - "file system problem"
owner: devops-agent
objective: "Provide a systematic approach to diagnosing unclassified FSx issues"
context: "This catch-all runbook covers FSx issues that don't clearly fit into specific categories. It provides a general diagnostic workflow applicable to all FSx file system types (Lustre, Windows, ONTAP, OpenZFS)."
---

## Phase 1 — Triage

MUST:
- Identify the file system type: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check file system Lifecycle status (AVAILABLE, CREATING, UPDATING, FAILED, DELETING, MISCONFIGURED)
- Check for recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`
- Verify basic connectivity (security group, subnet, DNS)

SHOULD:
- Check CloudWatch metrics for anomalies: `aws cloudwatch list-metrics --namespace AWS/FSx --dimensions Name=FileSystemId,Value=<fs-id>`
- Review administrative actions on the file system
- Check for service health issues in the AWS Health Dashboard
- Verify the file system is not undergoing maintenance

MAY:
- Check AWS service quotas for FSx
- Review recent configuration changes via CloudTrail
- Check for related service issues (AD, KMS, S3, VPC)

## Phase 2 — Remediate

MUST:
- Address the identified root cause based on triage findings
- Route to the appropriate specific runbook if the issue is identified
- Verify the file system returns to AVAILABLE state after remediation

SHOULD:
- Document the issue and resolution for future reference
- Set up monitoring to detect similar issues proactively
- Review file system configuration for best practices

MAY:
- Implement automated health checks
- Create CloudWatch dashboards for FSx monitoring

## Common Issues

- symptoms: "File system in MISCONFIGURED state"
  diagnosis: "A configuration dependency has changed (AD, subnet, KMS key)."
  resolution: "Check FailureDetails in describe-file-systems. Fix the identified dependency."

- symptoms: "Administrative action stuck in PENDING"
  diagnosis: "A previous operation has not completed or the file system is in a transitional state."
  resolution: "Wait for the current operation to complete. Contact AWS Support if stuck for extended period."

- symptoms: "Unexpected file system behavior after region event"
  diagnosis: "AWS service event may have affected the file system or its dependencies."
  resolution: "Check AWS Health Dashboard. Verify file system and dependency health. Contact AWS Support if needed."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: file_system
    content: "<file system status and configuration>"
  - type: cloudtrail
    content: "<recent API events>"
  - type: cloudwatch
    content: "<relevant metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified root cause"
  long_term: "Implement comprehensive monitoring and alerting"
```


---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — FSx provides shared file storage for compute workloads, databases, and analytics |
| Data Risk | CRITICAL — File systems contain production data, application state, and potentially irreplaceable datasets |
| Recovery Complexity | HIGH — File system recreation requires backup restoration and client remounting across all consumers |
| Blast Radius | HIGH — File system outages affect all connected compute instances and dependent applications |

## Escalation Conditions

- File system enters FAILED or MISCONFIGURED state with no clear remediation path
- Multi-AZ failover not completing or standby file server unavailable during AZ event
- Data corruption detected on file system volumes or inconsistent backup state
- KMS key disabled or deleted causing file system inaccessibility
- Storage capacity at 100% utilization with no ability to scale (quota limits reached)
- Backup or restore operations failing on business-critical file systems

## Data Sensitivity

- File systems may contain regulated data (financial records, healthcare data, PII)
- Backup contents mirror production data sensitivity classifications
- Active Directory integration credentials reveal domain architecture and trust relationships
- NFS export policies and NTFS ACLs expose organizational access control structure
- ONTAP SVM configurations reveal multi-tenant data isolation boundaries

## Prohibited Actions

- NEVER suggest deleting file systems without confirming backups exist and are restorable
- NEVER suggest disabling encryption or using unencrypted file systems for any workload
- NEVER suggest reducing storage capacity — this operation is irreversible on all FSx types
- NEVER modify Active Directory integration on production Windows FSx without maintenance window
- NEVER remove or modify KMS key policies without verifying all dependent file systems
- NEVER change security group rules that could disconnect active file system clients

## Phase 3 — Rollback

- If file system configuration was changed: revert settings and verify client connectivity is restored
- If storage scaling failed: wait for pending operations to complete, verify file system returns to AVAILABLE
- If backup was deleted: check AWS Backup vault for retained copies, recreate from most recent available backup
- If security group was modified: restore previous rules immediately and verify all client mounts are functional
- If KMS key was affected: re-enable key or cancel deletion, verify file system recovers automatically
- Document all rollback actions with file system ID, affected clients, and data integrity verification steps

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest removing AD integration without review"
  - "NEVER suggest public subnet placement"
