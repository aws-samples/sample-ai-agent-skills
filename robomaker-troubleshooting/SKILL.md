---
name: robomaker-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS RoboMaker problems
  by analyzing simulation jobs, robot applications, fleet management,
  ROS integration, world generation, deployment pipelines, and following
  structured runbooks. Activate when: simulation job failures, robot
  application build errors, fleet deployment issues, ROS connectivity
  problems, world generation failures, or the user says something is
  wrong with RoboMaker without naming specific symptoms.
compatibility: >
  Requires AWS CLI with RoboMaker, S3, IAM, CloudWatch, ECR, IoT,
  and Greengrass permissions.
---

# AWS RoboMaker Diagnostics

## When to use

Any AWS RoboMaker investigation — simulation jobs, robot applications, fleet management, ROS integration, world generation, or deployment.

## Investigation workflow

### Step 1 — Collect and triage

```
aws robomaker list-simulation-jobs --query 'simulationJobSummaries[*].{Arn:arn,Status:status,Name:name}'
version: "1.0.0"
last_updated: "2025-04-12"
aws robomaker list-robot-applications
aws robomaker list-robots --query 'robots[*].{Arn:arn,Name:name,Status:status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws robomaker list-fleets
```

### Step 2 — Domain deep dive

```
aws robomaker describe-simulation-job --job <job-arn>
aws robomaker describe-robot-application --application <app-arn>
aws robomaker describe-fleet --fleet <fleet-arn>
aws robomaker list-world-generation-jobs
```

### Step 3 — Detailed investigation

```
aws robomaker describe-world-generation-job --job <job-arn>
aws logs filter-log-events --log-group-name /aws/robomaker/SimulationJobs --start-time <epoch-ms>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=robomaker.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any RoboMaker issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `robomaker list-simulation-jobs` | List simulation jobs and status |
| `robomaker describe-simulation-job` | Get job details and failure reason |
| `robomaker list-robot-applications` | List robot applications |
| `robomaker list-robots` | List registered robots |
| `robomaker list-fleets` | List robot fleets |
| `robomaker list-world-generation-jobs` | List world generation jobs |
| CloudWatch Logs | Simulation and robot logs |

## Gotchas: AWS RoboMaker

- Simulation jobs run in isolated VPC environments. Network access must be explicitly configured.
- Robot applications must be built for the target architecture (X86_64 or ARM64).
- ROS version must match between simulation and robot applications (ROS1 Melodic/ROS2 Foxy).
- World generation uses WorldForge templates. Complex worlds take longer to generate.
- Fleet deployment uses AWS IoT Greengrass. Robots must have Greengrass Core installed.
- Simulation job duration has limits. Jobs timeout after the configured max duration.

## Anti-hallucination rules

1. Always cite simulation job ARNs, robot ARNs, or API responses as evidence.
2. RoboMaker simulation runs in AWS cloud, not on physical robots.
3. ROS version compatibility is strict. Never mix ROS1 and ROS2.
4. Fleet deployment requires Greengrass. Never suggest direct deployment.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Simulation | A1-A3 | Job failures, configuration, batch simulation |
| B — Robot Apps | B1-B2 | Build errors, runtime failures |
| C — Fleet | C1-C2 | Fleet management, robot registration |
| D — ROS | D1-D2 | ROS integration, topic connectivity |
| E — World Gen | E1-E2 | World generation, template issues |
| F — Deployment | F1-F2 | Greengrass deployment, update failures |
| Z — Catch-All | Z1-Z2 | General troubleshooting, IAM issues |
