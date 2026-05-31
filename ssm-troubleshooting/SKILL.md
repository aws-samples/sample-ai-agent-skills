---
name: ssm-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Systems Manager problems
  by analyzing SSM Agent connectivity, Session Manager, Run Command, Patch Manager,
  State Manager, Parameter Store, Automation, Inventory, Maintenance Windows,
  and following structured runbooks. Activate when: SSM Agent not connecting,
  managed instance registration failures, hybrid activation issues, session
  failures, port forwarding problems, command failures, command timeouts,
  targeting issues, patch baseline problems, patching failures, compliance
  gaps, association failures, drift detection, parameter access denied,
  throughput limits, parameter tier issues, runbook failures, approval
  workflow problems, inventory collection failures, custom inventory issues,
  maintenance window execution failures, task failures, or the user says
  something is wrong with AWS Systems Manager without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SSM, EC2, IAM, CloudTrail,
  CloudWatch, and optionally VPC, KMS, and S3 permissions. Some
  operations require managed instance IAM role with
  AmazonSSMManagedInstanceCore policy.
---

# AWS Systems Manager Diagnostics

## When to use

Any AWS Systems Manager investigation where the console alone is insufficient — SSM Agent connectivity, Session Manager troubleshooting, Run Command debugging, Patch Manager compliance, State Manager associations, Parameter Store access, Automation runbooks, Inventory collection, or Maintenance Window execution.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion,Platform:PlatformType}'
aws ssm describe-instance-information --filters Key=PingStatus,Values=ConnectionLost
aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,IAMRole:IamInstanceProfile.Arn,SubnetId:SubnetId}'
aws ssm get-connection-status --target <instance-id>
```

### Step 2 — Domain deep dive

```
aws ssm list-commands --filters key=Status,value=Failed --max-results 10
aws ssm describe-sessions --state Active --filters key=Target,value=<instance-id>
aws ssm describe-patch-baselines --filters Key=OWNER,Values=Self
aws ssm list-associations --query 'Associations[*].{Id:AssociationId,Name:Name,Status:Overview.Status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws ssm describe-parameters --parameter-filters Key=Name,Values=<param-name>
aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed
aws ssm describe-maintenance-windows --query 'WindowIdentities[*].{Id:WindowId,Name:Name,Enabled:Enabled}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 3 — Detailed investigation

```
aws ssm get-command-invocation --command-id <cmd-id> --instance-id <instance-id>
aws ssm describe-patch-group-state --patch-group <group>
aws ssm describe-effective-patches-for-patch-baseline --baseline-id <baseline-id>
aws ssm get-inventory --filters Key=AWS:InstanceInformation.InstanceId,Values=<instance-id>,Type=Equal
aws ssm describe-automation-step-executions --automation-execution-id <exec-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20
```

Read `references/ssm-guardrails.md` before concluding on any SSM issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-instance-information` | Check managed instance status and agent version |
| `get-connection-status` | Verify SSM connectivity for a specific instance |
| `list-commands` | List Run Command executions |
| `get-command-invocation` | Get command output for a specific instance |
| `describe-sessions` | List Session Manager sessions |
| `describe-patch-baselines` | List patch baselines |
| `describe-patch-group-state` | Patch compliance for a patch group |
| `list-associations` | List State Manager associations |
| `describe-parameters` | List/search parameters |
| `get-parameter` | Get parameter value |
| `describe-automation-executions` | List automation executions |
| `describe-maintenance-windows` | List maintenance windows |
| `get-inventory` | Get inventory data for instances |

## Gotchas: AWS Systems Manager

- SSM Agent must be installed AND have IAM role with AmazonSSMManagedInstanceCore. The agent alone is not enough — the instance must have an IAM instance profile with the AmazonSSMManagedInstanceCore managed policy (or equivalent permissions). Without this, the instance will not appear in Fleet Manager or respond to commands.
- Managed instances need outbound access to SSM endpoints (or VPC endpoints). SSM Agent communicates with ssm.<region>.amazonaws.com, ssmmessages.<region>.amazonaws.com, and ec2messages.<region>.amazonaws.com. Without outbound HTTPS (443) access to these endpoints (via NAT Gateway, internet gateway, or VPC endpoints), the agent cannot connect.
- Session Manager replaces SSH — no inbound ports needed. Session Manager provides shell access without opening inbound SSH ports. It uses the SSM Agent's outbound connection. This requires ssmmessages endpoint access and the Session Manager plugin installed locally for CLI usage.
- Run Command has a 2-hour timeout for interactive commands. The default timeout for Run Command is 3600 seconds (1 hour), configurable up to 172800 seconds (48 hours). However, interactive commands that produce no output for 2 hours are terminated. Long-running scripts should write periodic output.
- Parameter Store standard tier is free (4KB limit), advanced tier costs (8KB limit). Standard parameters are free up to 10,000 parameters with 4KB max size. Advanced parameters cost $0.05/month per parameter, support 8KB size, parameter policies (expiration, notification), and higher throughput. Throughput is 40 TPS standard, 1000 TPS with higher throughput enabled.
- Automation documents are YAML or JSON. SSM Automation documents (runbooks) can be authored in YAML or JSON. They support conditional branching, approval steps, and cross-account execution. Custom documents must be in the correct schema version (2.2 for Command, 0.3 for Automation).
- Patch baselines are OS-specific. Each patch baseline targets a specific operating system. You cannot use a Windows baseline for Linux instances. AWS provides default baselines per OS. Custom baselines allow fine-grained control over which patches are approved.
- Maintenance windows schedule tasks with cron or rate expressions. Windows use cron/rate expressions and have a duration and cutoff time. Tasks that haven't started by the cutoff are skipped. Multiple tasks can run in a single window with priority ordering.

## Anti-hallucination rules

1. Always cite specific instance IDs, command IDs, or API responses as evidence.
2. SSM Agent requires BOTH installation AND IAM role. Never suggest only one is sufficient.
3. VPC endpoint or outbound internet access is required. Never assume connectivity exists.
4. Parameter Store standard tier is free with 4KB limit. Never confuse tier limits.
5. Patch baselines are OS-specific. Never suggest cross-OS baseline usage.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Agent | A1-A3 | SSM Agent not connecting, managed instance registration, hybrid activation |
| B — Session Manager | B1-B2 | Session failures, port forwarding |
| C — Run Command | C1-C3 | Command failures, timeout, targeting |
| D — Patch Manager | D1-D3 | Patch baseline, patching failures, compliance |
| E — State Manager | E1-E2 | Association failures, drift |
| F — Parameter Store | F1-F3 | Parameter access, throughput, tiers |
| G — Automation | G1-G2 | Runbook failures, approval workflows |
| H — Inventory | H1-H2 | Collection failures, custom inventory |
| I — Maintenance Windows | I1-I2 | Window execution, task failures |
| Z — Catch-All | Z1 | General troubleshooting |
