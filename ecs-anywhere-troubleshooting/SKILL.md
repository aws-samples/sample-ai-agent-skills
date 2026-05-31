---
name: ecs-anywhere-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot ECS Anywhere
  problems by analyzing external instance registration, SSM agent
  issues, task placement failures, container runtime errors,
  networking, service discovery, IAM roles, credential management,
  monitoring, log collection, capacity providers, and following
  structured runbooks.
  Activate when: external instance registration failures, SSM agent
  issues, task placement errors, container runtime problems,
  networking issues, service discovery failures, IAM configuration
  errors, credential problems, monitoring gaps, log collection
  issues, capacity provider errors, or the user says something is
  wrong with ECS Anywhere.
compatibility: >
  Requires AWS CLI or SDK access with ecs, ssm, iam, cloudwatch,
  logs, ec2, and cloudtrail permissions. External instances require
  SSM Agent and Docker/containerd.
---

# ECS Anywhere Diagnostics

## When to use

Any ECS Anywhere investigation — external instance registration, SSM agent issues, task placement failures, container runtime errors, networking, service discovery, IAM roles, credential management, monitoring, log collection, or capacity provider configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ecs list-container-instances --cluster <cluster-name> --filter "attribute:ecs.os-type == linux" --status ACTIVE
aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn> --query 'containerInstances[*].{Id:ec2InstanceId,Status:status,Agent:agentConnected,RunningTasks:runningTasksCount}'
aws ssm describe-instance-information --filters Key=ResourceType,Values=ManagedInstance --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,Platform:PlatformName}'
```

### Step 2 — Domain deep dive

```
aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arn> --query 'tasks[*].{Status:lastStatus,StoppedReason:stoppedReason,Containers:containers[*].{Name:name,Status:lastStatus,ExitCode:exitCode}}'
aws ssm get-connection-status --target <managed-instance-id>
aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].{Status:status,DesiredCount:desiredCount,RunningCount:runningCount,Events:events[:5]}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ecs.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name /ecs/<cluster-name> --start-time <epoch-ms> --filter-pattern "ERROR"
aws ecs describe-capacity-providers --capacity-providers <provider-name>
```

Read `references/guardrails.md` before concluding on any ECS Anywhere issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `ecs list-container-instances` | List registered external instances |
| `ecs describe-container-instances` | Get instance details and status |
| `ssm describe-instance-information` | Check SSM agent connectivity |
| `ecs describe-tasks` | Get task status and stopped reasons |
| `ecs describe-services` | Get service status and events |
| `ssm get-connection-status` | Check SSM connection for an instance |
| `ecs describe-capacity-providers` | Get capacity provider configuration |

## Gotchas: ECS Anywhere

- External instances register as EXTERNAL launch type, not EC2 or FARGATE. Task definitions must specify EXTERNAL as a compatible launch type.
- SSM Agent is REQUIRED on all external instances. It handles registration, credential rotation, and communication with AWS. If SSM Agent stops, the instance becomes disconnected.
- External instances use SSM managed instances (mi-*) not EC2 instances (i-*). The instance ID format is different.
- ECS Agent on external instances gets temporary credentials via SSM. Credential rotation happens automatically but requires SSM Agent to be running.
- Networking on external instances uses the host network mode or bridge mode. awsvpc mode is NOT supported on external instances.
- Service discovery with Cloud Map works but requires the external instance to have network connectivity to the discovered endpoints.
- External instances do NOT support GPU task placement, Elastic Inference, or EFS volumes.

## Anti-hallucination rules

1. Always cite specific instance ARNs, task ARNs, or SSM instance IDs as evidence.
2. EXTERNAL launch type is distinct from EC2 and FARGATE. Never conflate them.
3. SSM Agent is mandatory for ECS Anywhere. Never suggest alternatives.
4. awsvpc network mode is NOT supported on external instances. Never recommend it.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Registration | A1-A2 | External instance registration, SSM agent issues |
| B — Task Execution | B1-B2 | Task placement failures, container runtime errors |
| C — Networking | C1-C2 | Networking issues, service discovery |
| D — IAM & Credentials | D1-D2 | IAM role configuration, credential management |
| E — Observability | E1-E2 | Monitoring gaps, log collection |
| F — Capacity | F1 | Capacity provider |
| Z — Catch-All | Z1 | General troubleshooting |
