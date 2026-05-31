---
name: bottlerocket-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Bottlerocket OS
  problems by analyzing AMI launches, OS updates, ECS/EKS agent
  issues, admin/control container access, networking, storage,
  settings API, bootstrap, security hardening, and following
  structured runbooks.
  Activate when: AMI launch failures, update failures, ECS agent
  issues, kubelet problems, admin container access, control
  container issues, networking misconfigurations, storage errors,
  settings API failures, bootstrap problems, security hardening
  questions, or the user says something is wrong with Bottlerocket.
compatibility: >
  Requires AWS CLI or SDK access with ec2, ecs, eks, ssm, s3,
  cloudwatch, and autoscaling permissions. SSH or SSM Session
  Manager access to Bottlerocket instances for host-level
  diagnostics.
---

# Bottlerocket Diagnostics

## When to use

Any Bottlerocket investigation — AMI launch failures, OS update issues, ECS agent problems, kubelet errors, admin/control container access, networking configuration, storage issues, settings API errors, bootstrap failures, or security hardening.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ec2 describe-instances --filters "Name=tag:OperatingSystem,Values=Bottlerocket" --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,ImageId:ImageId,LaunchTime:LaunchTime}'
aws ssm describe-instance-information --filters "Key=PlatformType,Values=Linux" --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,Platform:PlatformName,Version:PlatformVersion}'
aws ec2 describe-images --owners amazon --filters "Name=name,Values=bottlerocket-aws-k8s-*" --query 'Images | sort_by(@, &CreationDate) | [-5:].{Name:Name,ImageId:ImageId,Created:CreationDate}'
```

### Step 2 — Domain deep dive

```
aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings"
aws ecs describe-container-instances --cluster <cluster> --container-instances <arn> --query 'containerInstances[*].{Status:status,Agent:agentConnected,Version:versionInfo}'
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup> --query 'nodegroup.{Status:status,AmiType:amiType,Version:version}'
```

### Step 3 — Detailed investigation

```
aws ec2 get-console-output --instance-id <instance-id> --latest
aws cloudwatch get-metric-statistics --namespace AWS/EC2 --metric-name StatusCheckFailed --dimensions Name=InstanceId,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="logdog"
```

Read `references/guardrails.md` before concluding on any Bottlerocket issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ec2 describe-instances` | List Bottlerocket instances and status |
| `ec2 describe-images` | Find available Bottlerocket AMIs |
| `ec2 get-console-output` | Retrieve boot logs for diagnosis |
| `ssm start-session` | Access admin or control container |
| `ecs describe-container-instances` | Check ECS agent connectivity |
| `eks describe-nodegroup` | Check EKS node group health |
| `apiclient get settings` | Read Bottlerocket settings via API |

## Gotchas: Bottlerocket

- Bottlerocket has NO package manager and NO shell by default. The admin container must be enabled for SSH-like access. The control container provides SSM access.
- Bottlerocket uses an API-driven settings model (`apiclient`). Configuration files are NOT directly editable — all changes go through the settings API.
- Updates use a dual-partition (A/B) scheme. The inactive partition is updated, then the system reboots into it. Rollback swaps back to the previous partition.
- ECS and EKS variants use DIFFERENT AMIs. Using the wrong variant (e.g., k8s AMI for ECS) causes agent failures.
- The admin container is disabled by default for security. Enabling it requires user data or SSM. It provides `sheltie` to access the host filesystem.
- Bottlerocket logs are collected via `logdog` command in the control container, NOT via traditional `/var/log` paths.
- SELinux is enforced by default. Custom configurations that bypass SELinux will fail silently.

## Anti-hallucination rules

1. Always cite specific instance IDs, AMI IDs, or API responses as evidence.
2. ECS variant and EKS variant are completely different AMIs. Never suggest using one for the other.
3. Bottlerocket has NO yum, apt, or any package manager. Never suggest installing packages.
4. Settings are API-driven only. Never suggest editing config files directly on the host.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — AMI & Updates | A1–A2 | AMI launch failures, update failures |
| B — Container Agents | B1–B2 | ECS agent issues, kubelet issues |
| C — Access | C1–C2 | Admin container access, control container issues |
| D — Infrastructure | D1–D2 | Networking configuration, storage issues |
| E — Configuration | E1–E2 | Settings API errors, bootstrap failures |
| F — Security | F1 | Security hardening |
| Z — Catch-All | Z1 | General troubleshooting |
