---
name: ssm-fleet-manager-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager
  Fleet Manager problems by analyzing managed node registration,
  remote desktop, file system access, performance counters, OS patching
  visibility, hybrid activations, IAM instance profiles, SSM Agent
  connectivity, and account-level settings using structured runbooks.
  Activate when: managed nodes not appearing, remote desktop failures,
  file system access errors, performance counter issues, patching
  visibility problems, hybrid activation errors, IAM profile issues,
  SSM Agent connectivity problems, or the user says something is wrong
  with Fleet Manager.
compatibility: >
  Requires AWS CLI or SDK access with ssm, ec2, iam, cloudwatch,
  cloudtrail, and sts permissions.
---

# AWS Systems Manager Fleet Manager Diagnostics

## When to use

Any Fleet Manager investigation — managed node registration, node visibility, remote desktop, file system access, performance counters, OS patching visibility, hybrid activations, IAM instance profiles, SSM Agent connectivity, or account-level settings.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion,Platform:PlatformType}'
aws ssm describe-instance-associations-status --instance-id <instance-id>
aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,Profile:IamInstanceProfile.Arn}'
```

### Step 2 — Domain deep dive

```
aws ssm get-connection-status --target <instance-id>
aws ssm describe-activations --query 'ActivationList[*].{Id:ActivationId,Expired:Expired,RegistrationLimit:RegistrationLimit,RegistrationsCount:RegistrationsCount}'
aws iam get-instance-profile --instance-profile-name <profile-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/activation-tier
aws ssm describe-instance-patches --instance-id <instance-id> --query 'Patches[*].{KB:KBId,State:State,Severity:Severity}'
```

Read `references/guardrails.md` before concluding on any Fleet Manager issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ssm describe-instance-information` | List managed nodes and status |
| `ssm get-connection-status` | Check node connectivity |
| `ssm describe-activations` | List hybrid activations |
| `ssm start-session` | Start Session Manager session |
| `ssm send-command` | Run commands on nodes |
| `ssm describe-instance-patches` | Check patch compliance |
| `ssm get-service-setting` | Check account-level settings |

## Gotchas: Fleet Manager

- Fleet Manager requires SSM Agent 3.0+ for full functionality. Older agents support basic features but not remote desktop, file system, or performance counters.
- Managed nodes must have an IAM instance profile with `AmazonSSMManagedInstanceCore` policy. Missing or incorrect profiles are the #1 cause of nodes not appearing.
- Hybrid activations (on-premises servers) use `mi-` prefixed instance IDs, not `i-` prefixed. They require separate activation codes and have registration limits.
- Remote Desktop via Fleet Manager requires port 3389 (Windows) or NICE DCV for Linux. It does NOT open inbound security group rules — it tunnels through SSM.
- Performance counters are only available for Windows managed nodes. Linux nodes use CloudWatch Agent for equivalent metrics.
- Fleet Manager is regional. Nodes only appear in the region where their SSM Agent connects. Multi-region fleets require checking each region.
- The SSM Agent must be able to reach SSM endpoints. In VPCs without internet access, VPC endpoints for ssm, ssmmessages, and ec2messages are required.

## Anti-hallucination rules

1. Always cite specific instance IDs, activation IDs, or API responses as evidence.
2. Hybrid nodes (mi-*) and EC2 nodes (i-*) have different registration mechanisms. Never conflate them.
3. Fleet Manager remote desktop tunnels through SSM — it does NOT require inbound security group rules for RDP/SSH.
4. Performance counters are Windows-only. Never suggest them for Linux nodes.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Registration | A1–A2 | Managed node registration, node not showing |
| B — Remote Access | B1–B2 | Remote desktop failures, file system access errors |
| C — Monitoring | C1–C2 | Performance counters issues, OS patching visibility |
| D — Activation | D1–D2 | Managed instance activation, hybrid activation errors |
| E — Connectivity | E1–E2 | IAM instance profile, SSM Agent connectivity |
| F — Settings | F1 | Account-level settings |
| Z — Catch-All | Z1 | General troubleshooting |
