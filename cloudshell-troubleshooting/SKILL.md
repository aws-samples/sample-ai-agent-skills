---
name: cloudshell-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS CloudShell
  problems by analyzing session launches, environment persistence,
  network connectivity, storage, permissions, timeouts, performance,
  and following structured runbooks.
  Activate when: session launch failures, environment persistence issues,
  network connectivity problems, VPC environment errors, storage issues,
  file transfer failures, permissions errors, credential issues,
  timeout/disconnection problems, performance issues, custom environment
  problems, or the user says something is wrong with CloudShell.
compatibility: >
  Requires AWS CLI or SDK access with cloudshell, iam, vpc,
  cloudtrail, and cloudwatch permissions.
---

# AWS CloudShell Diagnostics

## When to use

Any AWS CloudShell investigation — session launch failures, environment persistence, network connectivity, VPC environments, storage issues, file transfer, permissions, credentials, timeouts, performance, or custom environment configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws cloudshell get-environment-status --environment-id <env-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cloudshell.amazonaws.com --max-results 20
aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names cloudshell:CreateEnvironment cloudshell:GetEnvironmentStatus
```

### Step 2 — Domain deep dive

```
aws cloudshell describe-environments
aws ec2 describe-vpcs --vpc-ids <vpc-id>
aws ec2 describe-subnets --subnet-ids <subnet-id> --query 'Subnets[*].{Id:SubnetId,Az:AvailabilityZone,Cidr:CidrBlock}'
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/CloudShell --metric-name SessionDuration --start-time <start> --end-time <end> --period 3600 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version>
```

Read `references/guardrails.md` before concluding on any CloudShell issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `cloudshell get-environment-status` | Check environment state |
| `cloudshell describe-environments` | List all environments |
| `cloudshell create-environment` | Create new environment |
| `cloudshell delete-environment` | Delete environment |
| `cloudshell start-environment` | Start stopped environment |
| `cloudshell stop-environment` | Stop running environment |
| `iam simulate-principal-policy` | Test CloudShell permissions |

## Gotchas: AWS CloudShell

- CloudShell provides 1 GB of persistent storage per region in the home directory. Files outside $HOME are ephemeral and lost on session end.
- Sessions time out after 20 minutes of inactivity. Long-running processes must handle reconnection or use screen/tmux.
- VPC environments require a VPC, subnet, and security group. They do NOT have internet access by default — you need a NAT gateway.
- CloudShell is NOT available in all AWS regions. Check regional availability before troubleshooting launch failures.
- Each user gets a maximum of 10 concurrent CloudShell environments across all regions.
- File upload limit is 1 GB per file. File download limit is 1 GB per file. Use S3 for larger transfers.
- CloudShell inherits the IAM permissions of the console user. There are no separate CloudShell-specific credentials.

## Anti-hallucination rules

1. Always cite specific environment IDs, error messages, or API responses as evidence.
2. Persistent storage is ONLY in $HOME (1 GB). Never claim other directories persist.
3. VPC environments have NO internet by default. Never assume internet connectivity in VPC mode.
4. CloudShell inherits console user permissions. Never suggest separate CloudShell IAM roles.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Sessions | A1-A2 | Session launch failures, environment persistence |
| B — Network | B1-B2 | Network connectivity, VPC environment |
| C — Storage | C1-C2 | Storage issues, file transfer |
| D — Permissions | D1-D2 | Permissions errors, credential issues |
| E — Connectivity | E1-E2 | Timeout/disconnection, performance issues |
| F — Customization | F1 | Custom environment |
| Z — Catch-All | Z1 | General troubleshooting |
