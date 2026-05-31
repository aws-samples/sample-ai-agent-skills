---
name: batch-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Batch problems
  by analyzing job failures, compute environments, job queues, job
  definitions, networking, array jobs, multi-node parallel jobs,
  performance, and security using structured runbooks. Activate when:
  job failures, jobs stuck in RUNNABLE, job timeout, compute environment
  failures, EC2 vs Fargate issues, spot interruptions, job queue
  scheduling problems, job definition errors, container configuration
  issues, VPC networking problems, ECR access failures, array job
  failures, multi-node parallel issues, throughput bottlenecks,
  resource optimization, IAM role errors, secrets access problems,
  or the user says something is wrong with AWS Batch without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Batch, EC2, ECS, CloudWatch,
  CloudTrail, ECR, and IAM permissions.
---

# AWS Batch Diagnostics

## When to use

Any AWS Batch investigation where the console alone is insufficient — job failures, compute environment issues, scheduling problems, container configuration, networking, array/multi-node jobs, performance optimization, or security troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws batch describe-compute-environments --compute-environments <ce-name>
aws batch describe-job-queues --job-queues <queue-name>
aws batch describe-jobs --jobs <job-id>
aws batch list-jobs --job-queue <queue-name> --job-status FAILED --max-results 10
aws batch list-jobs --job-queue <queue-name> --job-status RUNNABLE --max-results 10
```

### Step 2 — Domain deep dive

```
aws batch describe-job-definitions --job-definition-name <def-name> --status ACTIVE
aws batch describe-jobs --jobs <job-id> --query 'jobs[*].{Status:status,Reason:statusReason,Container:container.{ExitCode:exitCode,Reason:reason,LogStream:logStreamName}}'
aws logs get-log-events --log-group-name /aws/batch/job --log-stream-name <stream-name>
aws ec2 describe-instances --filters Name=tag:aws:batch:compute-environment,Values=<ce-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=batch.amazonaws.com --max-results 20
aws ecs describe-clusters --clusters <ecs-cluster-arn>
aws cloudwatch get-metric-statistics --namespace AWS/Batch --metric-name CPUUtilization --dimensions Name=JobQueue,Value=<queue-name> --start-time <start> --end-time <end> --period 300 --statistics Average
```

Read `references/batch-guardrails.md` before concluding on any AWS Batch issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-jobs` | Check job status, exit code, reason, log stream |
| `list-jobs` | List jobs by status in a queue |
| `describe-compute-environments` | Check CE config, state, instance types |
| `describe-job-queues` | Check queue priority and CE associations |
| `describe-job-definitions` | Check container config, resources, IAM |
| CloudWatch Logs | Check container stdout/stderr logs |
| ECS | Check underlying ECS cluster and tasks |
| EC2 | Check instances in managed CE |

## Gotchas: AWS Batch

- RUNNABLE status means waiting for resources, not running. Jobs in RUNNABLE are waiting for compute capacity. This can be caused by: CE not scaling, instance types unavailable, VPC/subnet issues, or resource requirements exceeding CE limits. RUNNABLE is the most common stuck state.
- EC2 and Fargate compute environments have different capabilities. EC2 supports GPUs, custom AMIs, placement groups, and more instance types. Fargate is serverless but has vCPU/memory limits and no GPU support. Fargate Spot is available for cost savings.
- Job exit codes indicate failure type. Exit code 0 = success. Non-zero = application failure. Exit code 137 = OOM killed (container exceeded memory). Exit code 1 = general application error. Check container logs for details.
- Spot interruptions can terminate running jobs. Spot instances can be reclaimed with 2-minute warning. Batch retries Spot-interrupted jobs automatically (if retry strategy configured). Use a mix of Spot and On-Demand for critical workloads.
- Array jobs share a single job definition. Each child job gets an AWS_BATCH_JOB_ARRAY_INDEX environment variable. Array size can be 2-10,000. Failed children don't affect other children. Dependencies can be set between array jobs.
- Multi-node parallel jobs require EC2 compute. They use ECS task placement for node coordination. The main node (index 0) coordinates work. All nodes must be able to communicate. Fargate does not support multi-node parallel.
- Job queue priority determines scheduling order. Higher priority queues are evaluated first. Jobs within a queue are FIFO. Fair-share scheduling can override priority-based scheduling. A queue can be associated with multiple CEs.
- Container resource requests must match CE capabilities. vCPU and memory requests must be available in the CE instance types. Requesting more resources than any instance type can provide causes RUNNABLE stuck. GPU requests require GPU instance types.

## Anti-hallucination rules

1. Always cite specific job IDs, exit codes, or API responses as evidence.
2. RUNNABLE means waiting for resources, NOT running. Never confuse the two.
3. Exit code 137 means OOM killed. Never ignore memory-related exit codes.
4. Fargate does not support GPUs or multi-node parallel. Never suggest these.
5. Spot interruptions are expected. Never treat them as unexpected failures.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Jobs | A1-A3 | Job failures, stuck RUNNABLE, timeout |
| B — Compute | B1-B3 | Compute environment failures, EC2 vs Fargate, spot interruptions |
| C — Queue | C1-C2 | Job queue issues, scheduling |
| D — Definition | D1-D2 | Job definition errors, container config |
| E — Networking | E1-E2 | VPC config, ECR access |
| F — Array/Multi | F1-F2 | Array jobs, multi-node parallel |
| G — Performance | G1-G2 | Throughput, resource optimization |
| H — Security | H1-H2 | IAM roles, secrets |
| Z — Catch-All | Z1 | General troubleshooting |
