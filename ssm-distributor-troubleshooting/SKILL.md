---
name: ssm-distributor-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager Distributor
  problems by analyzing package creation, version management, installation failures,
  association execution, cross-platform packaging, S3 storage, IAM permissions,
  and following structured runbooks.
  Activate when: package creation failures, version errors, installation issues,
  uninstallation problems, association execution failures, scheduling issues,
  cross-platform package errors, custom package problems, S3 storage issues,
  IAM permission errors, third-party package failures, or the user says
  something is wrong with SSM Distributor.
compatibility: >
  Requires AWS CLI or SDK access with ssm, s3, iam, ec2, cloudwatch,
  and cloudtrail permissions.
---

# AWS Systems Manager Distributor Diagnostics

## When to use

Any SSM Distributor investigation — package creation, version management, installation failures, uninstallation errors, association execution, scheduling, cross-platform packages, custom packages, S3 storage, IAM permissions, or third-party package issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm list-documents --document-filter-list key=DocumentType,value=Package --query 'DocumentIdentifiers[*].{Name:Name,Owner:Owner,PlatformTypes:PlatformTypes}'
aws ssm list-associations --query 'Associations[*].{AssociationId:AssociationId,Name:Name,Status:Overview.Status}'
aws ssm describe-instance-information --query 'InstanceInformationList[*].{InstanceId:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion,PlatformType:PlatformType}'
```

### Step 2 — Domain deep dive

```
aws ssm describe-document --name <package-name> --query '{Name:Name,Status:Status,DocumentVersion:DocumentVersion,PlatformTypes:PlatformTypes}'
aws ssm list-command-invocations --command-id <command-id> --details --query 'CommandInvocations[*].{InstanceId:InstanceId,Status:Status,StatusDetails:StatusDetails}'
aws ssm describe-association-executions --association-id <assoc-id> --query 'AssociationExecutions[*].{ExecutionId:ExecutionId,Status:Status,DetailedStatus:DetailedStatus}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
aws ssm get-command-invocation --command-id <command-id> --instance-id <instance-id> --query '{Status:Status,StandardOutputContent:StandardOutputContent,StandardErrorContent:StandardErrorContent}'
aws s3 ls s3://<bucket>/packages/<package-name>/ --recursive
```

Read `references/guardrails.md` before concluding on any SSM Distributor issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm list-documents --document-filter-list key=DocumentType,value=Package` | List Distributor packages |
| `ssm describe-document` | Get package details and versions |
| `ssm send-command` | Install/uninstall packages on instances |
| `ssm list-command-invocations` | Check command execution status |
| `ssm describe-association-executions` | Check association execution history |
| `ssm describe-instance-information` | Verify managed instance status |
| `s3 ls` | Check package artifacts in S3 |

## Gotchas: SSM Distributor

- Distributor packages are SSM Documents of type Package. They are NOT the same as regular SSM Documents (Command, Automation).
- Packages require platform-specific installers. A single package can contain installers for Windows, Linux, and macOS but each must be explicitly defined in the manifest.
- The SSM Agent must be running and have connectivity to SSM endpoints. Instances with stale PingStatus are NOT managed and cannot receive packages.
- Association-based installations run on a schedule. If the schedule is misconfigured, packages may install at unexpected times or not at all.
- Custom packages are stored in S3. The S3 bucket must be in the same region as the instances, and the SSM Agent needs s3:GetObject permission.
- Package versions are immutable once created. You cannot modify an existing version — you must create a new version.
- Third-party packages (e.g., AmazonCloudWatchAgent) are AWS-managed. You cannot modify them but can pin specific versions.

## Anti-hallucination rules

1. Always cite specific package names, command IDs, association IDs, or API responses as evidence.
2. Distributor packages are SSM Documents of type Package — never confuse with Command or Automation documents.
3. Package installation requires a running SSM Agent with active PingStatus. Never assume connectivity.
4. Custom packages require S3 storage in the same region. Never suggest cross-region S3 access for packages.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Packages | A1–A2 | Package creation failures, package version errors |
| B — Installation | B1–B2 | Installation failures, uninstallation errors |
| C — Associations | C1–C2 | Association execution failures, scheduling issues |
| D — Platform & Custom | D1–D2 | Cross-platform packages, custom package errors |
| E — Storage & IAM | E1–E2 | S3 package storage, IAM permissions |
| F — Third-Party | F1 | Third-party package issues |
| Z — Catch-All | Z1 | General troubleshooting |
