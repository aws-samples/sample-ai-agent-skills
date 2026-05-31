---
name: datasync-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS DataSync problems by
  analyzing task failures, agent deployment, location configuration, transfer
  performance, filtering, security, scheduling, and following structured
  runbooks. Activate when: task execution failures, tasks stuck in launching
  or queued state, verification errors, agent deployment issues, agent
  connectivity problems, S3 location errors, EFS/FSx location issues,
  NFS/SMB location failures, slow transfer speeds, bandwidth throttling,
  include/exclude filter mismatches, file metadata preservation issues,
  IAM role permission errors, encryption in transit problems, schedule
  configuration failures, incremental transfer issues, or the user says
  something is wrong with DataSync without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DataSync, S3, EFS, FSx, IAM,
  CloudWatch, CloudTrail, and EC2 permissions. Agent-based transfers
  require network access to the agent VM.
---

# AWS DataSync Diagnostics

## When to use

Any AWS DataSync investigation where the console alone is insufficient — task failures, agent issues, location configuration, transfer performance, filtering problems, security configuration, scheduling, or incremental transfer troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws datasync list-tasks --query 'Tasks[*].{Arn:TaskArn,Status:Status,Name:Name}'
version: "1.0.0"
last_updated: "2025-04-12"
aws datasync describe-task --task-arn <task-arn>
aws datasync describe-task-execution --task-execution-arn <task-execution-arn>
aws datasync list-agents --query 'Agents[*].{Arn:AgentArn,Name:Name,Status:Status}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws datasync describe-agent --agent-arn <agent-arn>
aws datasync describe-location-s3 --location-arn <location-arn>
aws datasync describe-location-nfs --location-arn <location-arn>
aws datasync describe-location-efs --location-arn <location-arn>
aws datasync describe-location-fsx-windows --location-arn <location-arn>
aws cloudwatch get-metric-statistics --namespace AWS/DataSync --metric-name BytesTransferred --dimensions Name=TaskId,Value=<task-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=datasync.amazonaws.com --max-results 20
aws datasync list-task-executions --task-arn <task-arn> --query 'TaskExecutions[*].{Arn:TaskExecutionArn,Status:Status}'
aws logs filter-log-events --log-group-name /aws/datasync --filter-pattern "ERROR"
```

Read `references/datasync-guardrails.md` before concluding on any DataSync issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-task` | Check task configuration, options, filters |
| `describe-task-execution` | Check execution status, results, bytes transferred |
| `describe-agent` | Check agent status, connectivity, VPC endpoint |
| `describe-location-*` | Check source/destination location configuration |
| `list-task-executions` | List all executions for a task |
| CloudWatch Metrics | Monitor transfer throughput, files transferred |
| CloudTrail | Audit API calls and configuration changes |
| CloudWatch Logs | Check DataSync task execution logs |

## Gotchas: AWS DataSync

- DataSync agents require outbound connectivity on ports 443 (HTTPS) and 80 (activation only). The agent communicates with the DataSync service over TLS. If using a VPC endpoint, the agent uses the VPC endpoint instead of public endpoints. Port 80 is only needed during initial activation.
- Task execution verification compares source and destination. Verification can be ONLY_FILES_TRANSFERRED (default), POINT_IN_TIME_CONSISTENT, or NONE. Full verification (POINT_IN_TIME_CONSISTENT) scans all files at source and destination, which can be slow for large datasets.
- DataSync preserves metadata by default but behavior varies by location type. POSIX metadata (uid, gid, permissions) is preserved for NFS/EFS/FSx for Lustre. NTFS metadata (ACLs, timestamps) is preserved for SMB/FSx for Windows. S3 stores metadata as object metadata. Not all metadata types transfer between all location types.
- Include/exclude filters use glob patterns, not regex. Filters are evaluated in order: excludes first, then includes. An empty include filter means include everything. Filters apply to the relative path from the source location subdirectory.
- Bandwidth throttling is per-task, not per-agent. The BytesPerSecond option limits the bandwidth for a single task execution. Multiple concurrent tasks on the same agent share the agent's network bandwidth. Set throttling to avoid saturating network links.
- DataSync creates a service-linked role automatically. The AWSServiceRoleForDataSyncDataAccess role is created when you first use DataSync. For S3 locations, you must provide a separate IAM role with S3 permissions. The S3 IAM role is NOT the same as the service-linked role.
- Incremental transfers only copy changed files. DataSync compares source and destination to identify changes. The comparison method depends on the PrepareStep and TransferStep options. Incremental transfers are the default behavior for subsequent task executions.
- Agent VM requires minimum 4 vCPUs and 16 GB RAM. Undersized agents cause poor performance or failures. The agent VM also needs at least 80 GB of disk space. For high-performance transfers, allocate more resources.
- Cross-region and cross-account transfers are supported but require proper IAM configuration. Cross-account transfers need IAM roles in both accounts. Cross-region transfers incur data transfer charges.

## Anti-hallucination rules

1. Always cite specific task ARNs, agent ARNs, or API responses as evidence.
2. DataSync agents are NOT the same as SSM agents. Never confuse the two.
3. Include/exclude filters use glob patterns, NOT regex. Never suggest regex syntax.
4. Verification modes have specific names. Never invent verification mode names.
5. Bandwidth throttling is per-task, not per-agent or per-account.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 22 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Task | A1–A3 | Task failures, task stuck, verification errors |
| B — Agent | B1–B2 | Agent deployment, agent connectivity |
| C — Location | C1–C3 | S3 location, EFS/FSx location, NFS/SMB location |
| D — Performance | D1–D2 | Transfer speed, bandwidth throttling |
| E — Filtering | E1–E2 | Include/exclude filters, file metadata |
| F — Security | F1–F2 | IAM roles, encryption in transit |
| G — Scheduling | G1–G2 | Schedule config, incremental transfers |
| Z — Catch-All | Z1 | General troubleshooting |
