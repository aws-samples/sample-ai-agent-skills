# AWS DataSync Diagnostics Skill

Agent skill for investigating and troubleshooting AWS DataSync problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS DataSync when the console alone isn't enough — task execution failures, agent deployment and connectivity, location configuration for S3/EFS/FSx/NFS/SMB, transfer performance optimization, include/exclude filtering, metadata preservation, IAM role configuration, encryption in transit, scheduling, and incremental transfers.

### Activate When

- Task execution failures or errors
- Task stuck in launching or queued state
- Verification errors after transfer
- Agent deployment or activation failures
- Agent connectivity or offline issues
- S3 location configuration errors
- EFS or FSx location issues
- NFS or SMB location connection failures
- Slow transfer speeds
- Bandwidth throttling configuration problems
- Include/exclude filter not matching expected files
- File metadata not preserved after transfer
- IAM role permission errors
- Encryption in transit configuration issues
- Schedule not triggering as expected
- Incremental transfer not detecting changes
- General DataSync errors without clear symptoms

---

## Skill Structure

```
datasync-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-task-failures.md
    ├── A2-task-stuck.md
    ├── A3-verification-errors.md
    ├── B1-agent-deployment.md
    ├── B2-agent-connectivity.md
    ├── C1-s3-location.md
    ├── C2-efs-fsx-location.md
    ├── C3-nfs-smb-location.md
    ├── D1-transfer-speed.md
    ├── D2-bandwidth-throttling.md
    ├── E1-include-exclude-filters.md
    ├── E2-file-metadata.md
    ├── F1-iam-roles.md
    ├── F2-encryption-in-transit.md
    ├── G1-schedule-config.md
    ├── G2-incremental-transfers.md
    ├── Z1-general-troubleshooting.md
    ├── datasync-guardrails.md
    └── datasync-hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Task** | A1–A3 | Task failures, task stuck, verification errors |
| **B — Agent** | B1–B2 | Agent deployment, agent connectivity |
| **C — Location** | C1–C3 | S3 location, EFS/FSx location, NFS/SMB location |
| **D — Performance** | D1–D2 | Transfer speed, bandwidth throttling |
| **E — Filtering** | E1–E2 | Include/exclude filters, file metadata |
| **F — Security** | F1–F2 | IAM roles, encryption in transit |
| **G — Scheduling** | G1–G2 | Schedule config, incremental transfers |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/datasync-guardrails.md` covering agent connectivity requirements, task verification modes, metadata preservation across location types, glob pattern filtering syntax, bandwidth throttling scope, service-linked role vs S3 IAM role distinction, incremental transfer behavior, agent VM sizing, cross-region/cross-account IAM, location subdirectory behavior, task execution lifecycle states, and NFS/SMB mount permissions.

---

## Investigation Workflow

1. **Triage** — Check task status, agent status, location configuration, recent executions
2. **Deep Dive** — Examine execution details, agent connectivity, location-specific settings
3. **Detailed** — Review CloudTrail events, CloudWatch metrics, task execution logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `datasync:Describe*`, `datasync:List*`, `s3:GetBucketLocation`, `efs:DescribeFileSystems`, `fsx:DescribeFileSystems`, `iam:GetRole`, `cloudtrail:LookupEvents`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents`
- Network access to DataSync agent VM for agent troubleshooting (optional)
- CloudWatch Logs enabled for DataSync task execution logging (recommended)

---

## Usage Examples

```
# List all DataSync tasks
aws datasync list-tasks

# Check task execution status
aws datasync describe-task-execution --task-execution-arn <task-execution-arn>

# Check agent status
aws datasync describe-agent --agent-arn <agent-arn>

# Check S3 location configuration
aws datasync describe-location-s3 --location-arn <location-arn>

# Monitor transfer throughput
aws cloudwatch get-metric-statistics --namespace AWS/DataSync --metric-name BytesTransferred --dimensions Name=TaskId,Value=<task-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

---

## License

MIT-0
