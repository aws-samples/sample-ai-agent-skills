---
name: elasticbeanstalk-diagnostics
description: >
  Use this skill to investigate and troubleshoot AWS Elastic Beanstalk problems
  by analyzing environments, deployments, health status, configuration,
  scaling, Docker platforms, worker tiers, and following structured runbooks.
  Activate when: environment creation failures, update failures, termination
  issues, platform version problems, deployment failures, rollback issues,
  immutable vs rolling deployment confusion, red/yellow health, health check
  misconfiguration, enhanced health reporting, .ebextensions errors,
  environment variable issues, instance profile problems, auto-scaling
  failures, load balancer configuration, Docker platform issues,
  multi-container Docker, worker tier SQS problems, dead letter queue
  issues, log retrieval failures, CloudWatch integration, or the user
  says something is wrong with Elastic Beanstalk without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Elastic Beanstalk, EC2, ELB,
  Auto Scaling, CloudFormation, S3, SQS, CloudWatch, IAM, and
  optionally RDS, SNS, and Route 53 permissions.
---

# AWS Elastic Beanstalk Diagnostics

## When to use

Any Elastic Beanstalk investigation where the console alone is insufficient — environment lifecycle issues, deployment debugging, health analysis, configuration troubleshooting, scaling problems, Docker platform issues, worker tier debugging, or logging and monitoring problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws elasticbeanstalk describe-environments --environment-names <env-name>
aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All
aws elasticbeanstalk describe-instances-health --environment-name <env-name>
aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20
aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name>
aws cloudformation describe-stack-events --stack-name <eb-stack-name> --max-items 20
```

### Step 2 — Domain deep dive

```
aws elasticbeanstalk describe-environment-resources --environment-name <env-name>
aws elasticbeanstalk list-platform-versions --filters "Type=PlatformOwner,Operator==,Values=AmazonWebServices"
aws elasticbeanstalk describe-platform-version --platform-arn <platform-arn>
aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type tail
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>
aws elbv2 describe-target-health --target-group-arn <tg-arn>
```

### Step 3 — Detailed investigation

```
aws elasticbeanstalk describe-configuration-options --application-name <app-name> --environment-name <env-name>
aws elasticbeanstalk validate-configuration-settings --application-name <app-name> --environment-name <env-name> --option-settings <settings>
aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All
aws iam get-instance-profile --instance-profile-name <profile-name>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticbeanstalk.amazonaws.com --max-results 20
aws logs describe-log-groups --log-group-name-prefix /aws/elasticbeanstalk/<env-name>
```

Read `references/elasticbeanstalk-guardrails.md` before concluding on any Elastic Beanstalk issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-environments` | Environment status, health, platform version |
| `describe-environment-health` | Detailed health with causes and metrics |
| `describe-instances-health` | Per-instance health status |
| `describe-events` | Environment event log with severity filtering |
| `describe-configuration-settings` | Full environment configuration |
| `describe-environment-resources` | ASG, ELB, instances, queues |
| `list-platform-versions` | Available platform branches and versions |
| `retrieve-environment-info` | Tail logs or bundle logs |
| `validate-configuration-settings` | Validate config before applying |
| `describe-configuration-options` | Available config options and constraints |
| `request-environment-info` | Request log bundle generation |
| `list-available-solution-stacks` | Available solution stacks |
| `describe-account-attributes` | Account-level EB quotas |
| `create-environment` | Create new environment |
| `update-environment` | Update environment configuration |
| `rebuild-environment` | Rebuild environment from scratch |

## Gotchas: AWS Elastic Beanstalk

- .ebextensions execute in alphabetical order. Files in the .ebextensions directory are processed in lexicographic order (01_packages.config before 02_services.config). If one extension depends on another, naming order matters. A failure in any extension fails the entire deployment.
- Environment rebuild vs update: rebuild terminates all instances and recreates the environment from scratch. Update modifies the existing environment in place. Rebuild loses all instance data and resets the environment. Use update for configuration changes; rebuild only when the environment is corrupted.
- Platform branch vs version: a platform branch (e.g., "Python 3.9 running on 64bit Amazon Linux 2") is a family of versions. A platform version is a specific release within a branch. Managed platform updates move between versions within the same branch. Changing branches requires a blue/green deployment or environment rebuild.
- Immutable deployment creates a new ASG. During immutable deployments, Beanstalk creates a temporary Auto Scaling group with new instances. If the health check passes, the new instances replace the old ones. If it fails, the temporary ASG is terminated. This means immutable deployments temporarily double your instance count and cost.
- Worker tier uses SQS, not direct HTTP. Worker environments poll an SQS queue for messages and POST them to localhost. They do NOT receive direct HTTP traffic. The daemon handles message retrieval, retry, and dead-letter queue routing. Misconfiguring the SQS queue URL or permissions is a common failure.
- Enhanced health requires the health agent. Enhanced health reporting requires the Elastic Beanstalk health agent running on instances. It is installed automatically on Amazon Linux 2 platforms but may be missing on older platforms. Without the agent, only basic health (ELB-based) is available.
- Managed platform updates can break apps. Managed updates automatically apply platform version updates during a maintenance window. If the new version introduces breaking changes (e.g., runtime version bump), the application may fail. Always test platform updates in a staging environment first.
- Environment cloning doesn't copy data. Cloning creates a new environment with the same configuration but does NOT copy RDS data, S3 objects, or any persistent state. If the environment has a coupled RDS instance, the clone gets a new empty database.
- Docker platform has different config than non-Docker. Docker environments use Dockerrun.aws.json (v1 for single, v3 for ECS-based multi-container). Non-Docker environments use Procfile and platform hooks. Mixing configuration approaches causes deployment failures.
- leader_only runs on one instance only. Commands marked with leader_only in .ebextensions run on only one instance (the leader). This is useful for database migrations but can cause issues if the leader instance is replaced. The leader is the first instance launched in the ASG.

## Anti-hallucination rules

1. Always cite specific environment names, event messages, or API responses as evidence.
2. .ebextensions execute in alphabetical order. Never suggest they run in parallel or random order.
3. Immutable deployments create a temporary ASG. Never suggest they modify instances in place.
4. Worker tier uses SQS polling, not direct HTTP. Never suggest sending HTTP requests to worker environments.
5. Enhanced health requires the health agent on instances. Never suggest enhanced health works without it.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Environment | A1-A4 | Creation failures, update failures, termination issues, platform version |
| B — Deployment | B1-B3 | Deployment failures, rollback, immutable vs rolling |
| C — Health | C1-C3 | Red/yellow health, health check config, enhanced health |
| D — Configuration | D1-D3 | .ebextensions errors, environment variables, instance profile |
| E — Scaling | E1-E2 | Auto-scaling issues, load balancer config |
| F — Docker | F1-F2 | Docker platform issues, multi-container |
| G — Worker | G1-G2 | Worker tier SQS, dead letter queue |
| H — Logging | H1-H2 | Log retrieval, CloudWatch integration |
| Z — Catch-All | Z1 | General troubleshooting |
