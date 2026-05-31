---
name: deadline-cloud-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Deadline Cloud
  problems by analyzing farm creation, fleet configuration, queues,
  job submission, worker agents, worker hosts, storage profiles,
  job attachments, budgets, license endpoints, monitors, identity
  management, custom job templates, and following structured runbooks.
  Activate when: farm creation failures, fleet configuration issues,
  queue problems, job submission errors, worker agent failures,
  worker host issues, storage profile errors, job attachment problems,
  budget configuration issues, license endpoint errors, monitor
  configuration failures, identity management problems, custom job
  template errors, or the user says something is wrong with Deadline Cloud.
compatibility: >
  Requires AWS CLI or SDK access with deadline, ec2, s3, iam,
  logs, and cloudwatch permissions.
---

# AWS Deadline Cloud Diagnostics

## When to use

Any AWS Deadline Cloud investigation — farm creation, fleet configuration, queue management, job submission, worker agents, worker hosts, storage profiles, job attachments, budgets, license endpoints, monitors, identity management, or custom job templates.

## Investigation workflow

### Step 1 — Collect and triage

```
aws deadline list-farms --query 'farms[*].{FarmId:farmId,DisplayName:displayName,KmsKeyArn:kmsKeyArn}'
aws deadline list-fleets --farm-id <farm-id> --query 'fleets[*].{FleetId:fleetId,DisplayName:displayName,Status:status,MaxWorkerCount:maxWorkerCount}'
aws deadline list-queues --farm-id <farm-id> --query 'queues[*].{QueueId:queueId,DisplayName:displayName,Status:status}'
```

### Step 2 — Domain deep dive

```
aws deadline get-farm --farm-id <farm-id>
aws deadline get-fleet --farm-id <farm-id> --fleet-id <fleet-id>
aws deadline get-queue --farm-id <farm-id> --queue-id <queue-id>
```

### Step 3 — Detailed investigation

```
aws deadline list-jobs --farm-id <farm-id> --queue-id <queue-id> --query 'jobs[*].{JobId:jobId,Name:name,LifecycleStatus:lifecycleStatus,TaskRunStatus:taskRunStatus}'
aws deadline list-workers --farm-id <farm-id> --fleet-id <fleet-id> --query 'workers[*].{WorkerId:workerId,Status:status,HostProperties:hostProperties}'
aws deadline list-sessions --farm-id <farm-id> --queue-id <queue-id> --job-id <job-id>
```

Read `references/guardrails.md` before concluding on any Deadline Cloud issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `deadline list-farms` | List all farms |
| `deadline get-farm` | Get farm details |
| `deadline list-fleets` | List fleets in a farm |
| `deadline get-fleet` | Get fleet configuration |
| `deadline list-queues` | List queues in a farm |
| `deadline list-jobs` | List jobs in a queue |
| `deadline list-workers` | List workers in a fleet |

## Gotchas: AWS Deadline Cloud

- Deadline Cloud has a hierarchy: Farm → Fleet/Queue → Job → Task → Session. Farms are the top-level container. Fleets provide compute. Queues organize jobs.
- Fleets can be customer-managed (your EC2 instances) or service-managed (AWS manages scaling). Service-managed fleets auto-scale based on job demand.
- Worker agents run on fleet instances and communicate with the Deadline Cloud service. Agent failures prevent job execution even if instances are healthy.
- Job attachments use S3 for file transfer between submitter and workers. Storage profiles define file system mappings. Misconfigured profiles cause job failures.
- Budgets control spending but do NOT stop running jobs. When a budget is exhausted, new jobs are blocked but existing jobs continue to completion.
- License endpoints connect to license servers for commercial DCC applications. Network connectivity between workers and license servers is critical.
- Custom job templates define job structure using OpenJD specification. Template errors cause job submission failures, not runtime failures.

## Anti-hallucination rules

1. Always cite specific farm IDs, fleet IDs, queue IDs, or API responses as evidence.
2. Service-managed and customer-managed fleets have different troubleshooting paths. Never conflate them.
3. Worker agent issues and worker host issues are different. Agent is software; host is infrastructure.
4. Job attachments use S3, not EFS or FSx by default. Never assume shared file system without checking storage profiles.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Farm & Fleet | A1-A2 | Farm creation, fleet configuration |
| B — Queue & Jobs | B1-B2 | Queue issues, job submission errors |
| C — Workers | C1-C2 | Worker agent failures, worker host issues |
| D — Storage | D1-D2 | Storage profiles, job attachments |
| E — Budget & License | E1-E2 | Budget configuration, license endpoints |
| F — Monitor & Identity | F1-F2 | Monitor configuration, identity management |
| G — Templates | G1 | Custom job templates |
| Z — Catch-All | Z1 | General troubleshooting |
