---
title: "H2 — Multi-AZ Failover Issues"
description: "Diagnose multi-AZ failover problems on FSx for Windows File Server and ONTAP"
status: active
severity: CRITICAL
triggers:
  - "failover failed"
  - "multi-AZ issue"
  - "standby not available"
  - "failover timeout"
  - "AZ failure"
  - "floating IP"
owner: devops-agent
objective: "Identify and resolve multi-AZ failover issues on FSx file systems"
context: "Multi-AZ is supported by Windows FSx and ONTAP. Windows uses active-standby with floating IP. ONTAP uses active-standby with automatic failover. Failover issues can stem from subnet configuration, DNS propagation, security group rules, or application reconnection logic."
---

## Phase 1 — Triage

MUST:
- Describe the file system for multi-AZ configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Verify the file system is multi-AZ (check SubnetIds has two subnets)
- Check file system Lifecycle status for failover state
- Check CloudWatch for failover events: `aws cloudwatch get-metric-statistics --namespace AWS/FSx --metric-name FileServerUnavailable --dimensions Name=FileSystemId,Value=<fs-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Check CloudTrail for failover events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=fsx.amazonaws.com --max-results 20`
- Verify both subnets are healthy: `aws ec2 describe-subnets --subnet-ids <subnet-1> <subnet-2>`
- Check DNS propagation for the file system DNS name
- Verify security groups allow traffic in both subnets

MAY:
- Check VPC route tables for both subnets
- Verify application reconnection behavior after failover
- Review failover history and duration

## Phase 2 — Remediate

MUST:
- Ensure both subnets have proper security group rules and routing
- Verify DNS resolution updates after failover (TTL should be low)
- Confirm applications handle brief disconnection during failover

SHOULD:
- Test failover procedures periodically
- Configure applications with retry logic for brief disconnections
- Monitor failover duration and frequency

MAY:
- Set up CloudWatch alarms for FileServerUnavailable metric
- Document failover procedures and expected behavior

## Common Issues

- symptoms: "Applications lose connectivity during failover and don't reconnect"
  diagnosis: "Application does not handle DNS changes or TCP connection reset after failover."
  resolution: "Implement retry logic in applications. Use short DNS TTL. Handle connection reset gracefully."

- symptoms: "Failover takes longer than expected (>60 seconds)"
  diagnosis: "DNS propagation delay or client DNS caching extends effective failover time."
  resolution: "Reduce DNS TTL on clients. Flush DNS cache after failover. Use file system IP directly if needed."

- symptoms: "Standby file server not available after primary failure"
  diagnosis: "Standby subnet has networking issues (security group, NACL, or route table)."
  resolution: "Verify standby subnet has identical networking configuration as the primary subnet."

## Output Format

```yaml
root_cause: "multi_az_failover — <specific_cause>"
evidence:
  - type: file_system
    content: "<multi-AZ configuration and failover status>"
  - type: subnets
    content: "<both subnet configurations and health>"
  - type: dns
    content: "<DNS resolution and propagation status>"
severity: CRITICAL
mitigation:
  immediate: "Fix subnet configuration or DNS propagation"
  long_term: "Test failover regularly and ensure applications handle reconnection"
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
