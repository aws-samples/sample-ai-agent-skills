---
name: backup-gateway-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Backup Gateway
  problems by analyzing gateway deployment, connectivity, hypervisor
  connections, vCenter authentication, VM discovery, VM backups,
  restores, bandwidth throttling, network configuration, and gateway
  maintenance.
  Activate when: gateway deployment failures, connectivity issues,
  hypervisor errors, VM backup failures, restore problems, performance
  issues, or the user says something is wrong with Backup Gateway.
compatibility: >
  Requires AWS CLI or SDK access with backup-gateway, backup, ec2,
  iam, cloudwatch, and cloudtrail permissions.
---

# AWS Backup Gateway Diagnostics

## When to use

Any AWS Backup Gateway investigation — gateway deployment, connectivity, hypervisor connections, vCenter authentication, VM discovery, VM backup errors, restore failures, restore performance, bandwidth throttling, network configuration, or gateway maintenance.

## Investigation workflow

### Step 1 — Collect and triage

```
aws backup-gateway list-gateways --query 'Gateways[*].{Name:GatewayDisplayName,Arn:GatewayArn,Type:GatewayType,LastSeen:LastSeenTime}'
aws backup-gateway list-hypervisors --query 'Hypervisors[*].{Name:Name,Arn:HypervisorArn,Host:Host,State:State}'
aws backup-gateway list-virtual-machines --query 'VirtualMachines[*].{Name:Name,Arn:ResourceArn,Path:Path,HypervisorId:HypervisorId}'
```

### Step 2 — Domain deep dive

```
aws backup-gateway get-gateway --gateway-arn <gateway-arn>
aws backup-gateway get-hypervisor --hypervisor-arn <hypervisor-arn>
aws backup list-backup-jobs --by-resource-type VIRTUALMACHINE --by-state FAILED --max-results 10
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=backup-gateway.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/BackupGateway --metric-name GatewayBytesUploaded --dimensions Name=GatewayArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws backup describe-backup-job --backup-job-id <job-id>
```

Read `references/guardrails.md` before concluding on any Backup Gateway issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `backup-gateway list-gateways` | List all backup gateways |
| `backup-gateway get-gateway` | Get gateway details and status |
| `backup-gateway list-hypervisors` | List connected hypervisors |
| `backup-gateway get-hypervisor` | Get hypervisor connection details |
| `backup-gateway list-virtual-machines` | List discovered VMs |
| `backup list-backup-jobs` | List backup job history |
| `backup describe-backup-job` | Get backup job details |

## Gotchas: AWS Backup Gateway

- Backup Gateway is a virtual appliance deployed on-premises in the VMware environment. It requires network connectivity to both vCenter/ESXi and AWS endpoints.
- The gateway must be able to reach AWS endpoints on port 443. If using a proxy, the gateway must be configured with proxy settings during deployment.
- vCenter credentials stored in the gateway are encrypted. If vCenter passwords change, the hypervisor connection must be updated with new credentials.
- VM discovery is periodic, not real-time. New VMs may take time to appear in the discovered VM list.
- Backup Gateway uses Changed Block Tracking (CBT) for incremental backups. If CBT is disabled or reset on a VM, the next backup will be a full backup.
- Restore creates a new VM — it does NOT overwrite the original. The restored VM needs network and storage configuration in vCenter.
- Gateway bandwidth is shared across all concurrent backup and restore operations. Multiple simultaneous operations compete for bandwidth.

## Anti-hallucination rules

1. Always cite specific gateway ARNs, hypervisor ARNs, or backup job IDs as evidence.
2. On-premises gateway and AWS cloud components have different failure modes. Never conflate them.
3. VM backup failures can be gateway-side, hypervisor-side, or AWS-side. Always identify the failure point.
4. Restore creates a NEW VM. Never claim restore overwrites the original VM.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Gateway | A1–A2 | Deployment failures, connectivity |
| B — Hypervisor | B1–B2 | Connection errors, vCenter auth |
| C — VMs | C1–C2 | Discovery failures, backup errors |
| D — Restore | D1–D2 | Restore failures, restore performance |
| E — Network | E1–E2 | Bandwidth throttling, network config |
| F — Maintenance | F1 | Gateway maintenance |
| Z — Catch-All | Z1 | General troubleshooting |
