---
name: ecs-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon ECS problems
  by analyzing service events, task state, container logs, and
  following structured runbooks. Activate when: tasks fail to start
  or keep restarting, services can't reach desired count, containers
  are OOM-killed, task definitions have errors, images can't be
  pulled, load balancer health checks fail, service discovery doesn't
  work, tasks can't reach VPC resources or the internet, IAM
  permissions are denied, capacity providers can't scale, deployments
  are stuck, exec into containers fails, or the user says something
  is wrong with ECS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ECS, CloudWatch Logs, EC2 (for
  EC2 launch type), ECR, ELB, IAM, and optionally VPC permissions.
---

# ECS Diagnostics

## When to use

Any ECS investigation where the ECS console alone is insufficient — stopped task reasons, container exit codes, OOM kills, image pull errors, networking issues, IAM permission failures, capacity provider scaling, deployment circuit breakers, service mesh issues, or exec failures.

## Investigation workflow

### Step 1 — Collect and triage

```
# Get service details and events
aws ecs describe-services --cluster <cluster> --services <service>

# Get recent task failures
aws ecs list-tasks --cluster <cluster> --service-name <service> --desired-status STOPPED
aws ecs describe-tasks --cluster <cluster> --tasks <task-arns>

# Get container logs
aws logs filter-log-events --log-group-name <log-group> --filter-pattern "ERROR"

# Get service events (last 100)
aws ecs describe-services --cluster <cluster> --services <service>  # → events[]
```

### Step 2 — Domain deep dive

```
# Task definition review
aws ecs describe-task-definition --task-definition <family:revision>

# Container instance state (EC2 launch type)
aws ecs list-container-instances --cluster <cluster>
aws ecs describe-container-instances --cluster <cluster> --container-instances <arns>

# Capacity providers
aws ecs describe-capacity-providers --capacity-providers <names>

# Load balancer health
aws elbv2 describe-target-health --target-group-arn <arn>

# ECR image check
aws ecr describe-images --repository-name <repo> --image-ids imageTag=<tag>
```

Read `references/ecs-guardrails.md` before concluding on any ECS issue.

### Step 3 — Detailed path

```
# ECS Exec for live debugging
aws ecs execute-command --cluster <cluster> --task <task-id> --container <name> --interactive --command "/bin/sh"

# CloudTrail for API events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunTask

# X-Ray traces (if enabled)
aws xray get-trace-summaries --start-time <time> --end-time <time>
```

## Gotchas: ECS

- Fargate tasks get a public IP only if assignPublicIp=ENABLED in awsvpcConfiguration. Without it, they need NAT for internet/ECR access.
- EC2 launch type: container instance must have ECS Agent running, IAM role with AmazonEC2ContainerServiceforEC2Role, and available CPU/memory.
- Task role (taskRoleArn) is for the APPLICATION inside the container. Execution role (executionRoleArn) is for ECS AGENT operations (pulling images, pushing logs, getting secrets).
- OOM kills: if a container exceeds its hard memory limit, ECS kills it (exit code 137). Soft limit (memoryReservation) is for scheduling only.
- Exit code 137 = SIGKILL (OOM or manual stop). Exit code 1 = application error. Exit code 139 = SIGSEGV. Exit code 143 = SIGTERM (graceful stop).
- Image pull failures: check ECR permissions on execution role, VPC endpoint or NAT for ECR access, and image tag existence.
- Service events show the last 100 events. Older events are lost. Check immediately when investigating.
- Deployment circuit breaker rolls back automatically after consecutive failures. Check deployment configuration.
- ECS Exec requires: SSM plugin, task role with ssmmessages permissions, enableExecuteCommand=true on service, and platform version 1.4.0+ (Fargate).
- awsvpc network mode: each task gets its own ENI. Subject to ENI limits per instance (EC2 launch type). Use ENI trunking for higher density.
- Service discovery (Cloud Map): health checks are separate from ELB health checks. Both must pass for traffic to flow.
- Fargate platform versions matter. 1.4.0 added EFS, ephemeral storage config, ECS Exec. Always use LATEST or specific version.

### Launch type comparison

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Infrastructure | Managed | Customer-managed |
| Networking | awsvpc only | awsvpc, bridge, host, none |
| Scaling | Task-level | Instance + task level |
| Pricing | Per vCPU/memory/second | EC2 instance cost |
| ECS Exec | Platform 1.4.0+ | Any |
| GPU | Not supported | Supported |

### Task stopped reasons

| Reason | Meaning |
|--------|---------|
| Essential container exited | A container marked essential stopped |
| OutOfMemoryError | Container exceeded hard memory limit |
| CannotPullContainerError | Image pull failed (auth, network, or missing image) |
| ResourceNotFoundException | Task definition or cluster not found |
| Scaling activity initiated | Service scaled down |

## Anti-hallucination rules

1. Always cite specific service events, stopped task reasons, or container exit codes as evidence.
2. Task role ≠ execution role. Never confuse which role is used for what.
3. Fargate tasks need NAT or VPC endpoints for internet/ECR access. Never claim Fargate has internet by default.
4. Exit code 137 is OOM or SIGKILL, not always OOM. Check the stoppedReason.
5. awsvpc mode means one ENI per task. ENI limits apply on EC2 launch type.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 38 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Task Failures | A1-A4 | Essential container exit, OOM kill, runtime crash, health check failure |
| B — Image Pull | B1-B3 | ECR auth, network/VPC endpoint, image not found |
| C — Networking | C1-C4 | No internet (Fargate), ENI limits, security group, service discovery |
| D — IAM/Permissions | D1-D4 | Execution role, task role, ECR pull, Secrets Manager/SSM |
| E — Deployment | E1-E3 | Stuck deployment, circuit breaker rollback, capacity provider |
| F — Performance | F1-F3 | CPU throttling, memory pressure, I/O bottleneck |
| G — Load Balancer | G1-G3 | Health check failure, target registration, deregistration delay |
| H — Capacity | H1-H3 | No EC2 capacity, Fargate capacity, resource reservation |
| I — Exec / Debug | I1-I2 | ECS Exec failures, log driver issues |
| J — Scheduling | J1-J2 | Placement constraints, attribute matching |
| Z — Catch-All | Z1 | General troubleshooting |
