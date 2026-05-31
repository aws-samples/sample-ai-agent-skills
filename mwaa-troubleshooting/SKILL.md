---
name: mwaa-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Managed Workflows for
  Apache Airflow (MWAA) problems by analyzing environment configurations, DAG
  import errors, task failures, scheduling issues, VPC networking, web server
  access, worker scaling, plugin and requirements issues, CloudWatch logging,
  IAM execution roles, secrets backend, S3 DAG bucket sync, and Airflow
  connections using structured runbooks. Activate when: environment creation or
  update failures, version upgrade issues, DAG import errors, task execution
  failures, scheduling problems, VPC networking issues, web server inaccessible,
  worker auto-scaling problems, task queue backlog, custom plugin failures,
  requirements.txt installation errors, CloudWatch log delivery issues, task log
  missing, IAM execution role permission errors, secrets backend configuration
  problems, S3 DAG bucket sync failures, Airflow connection issues, or the user
  says something is wrong with MWAA without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MWAA, S3, IAM, CloudWatch Logs, EC2 (VPC),
  CloudTrail, SecretsManager, and optionally KMS permissions. Airflow CLI access
  via MWAA CLI token for DAG and task operations.
---

# Amazon MWAA Diagnostics

## When to use

Any Amazon MWAA investigation where the console alone is insufficient — environment health analysis, DAG import debugging, task failure troubleshooting, scheduling issues, VPC networking, web server access, worker scaling, plugin/requirements problems, logging issues, IAM permissions, secrets backend, or S3 bucket sync.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mwaa list-environments
aws mwaa get-environment --name <env-name>
aws mwaa get-environment --name <env-name> --query 'Environment.Status'
aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,LastUpdate:LastUpdate}'
aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name RunningTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 300 --statistics Average
aws logs describe-log-groups --log-group-name-prefix airflow-<env-name>
```

### Step 2 — Domain deep dive

```
aws mwaa get-environment --name <env-name> --query 'Environment.{DagS3Path:DagS3Path,SourceBucketArn:SourceBucketArn,PluginsS3Path:PluginsS3Path,RequirementsS3Path:RequirementsS3Path}'
aws mwaa get-environment --name <env-name> --query 'Environment.ExecutionRoleArn'
aws iam get-role --role-name <execution-role-name>
aws iam list-attached-role-policies --role-name <execution-role-name>
aws s3 ls s3://<bucket-name>/<dags-path>/
aws mwaa create-cli-token --name <env-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=airflow.amazonaws.com --max-results 20
aws logs filter-log-events --log-group-name airflow-<env-name>-DAGProcessing --start-time <epoch-ms> --filter-pattern "ERROR"
aws logs filter-log-events --log-group-name airflow-<env-name>-Task --start-time <epoch-ms> --filter-pattern "ERROR"
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-subnets --subnet-ids <subnet-ids>
aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id>
```

Read `references/guardrails.md` before concluding on any MWAA issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `mwaa list-environments` | List all MWAA environments |
| `mwaa get-environment` | Full environment details, status, config |
| `mwaa create-cli-token` | Get CLI token for Airflow CLI commands |
| `s3 ls` | Check DAG files in S3 bucket |
| `logs filter-log-events` | Search scheduler, worker, DAG processing logs |
| `iam get-role` | Execution role details |
| `iam list-attached-role-policies` | Policies on execution role |
| `cloudtrail lookup-events` | Recent MWAA API events |
| `ec2 describe-security-groups` | VPC security group rules |
| `ec2 describe-nat-gateways` | NAT gateway for internet access |
| `secretsmanager list-secrets` | Secrets backend entries |
| `Airflow CLI (via token)` | DAG list, task state, trigger, connections |

## Gotchas: Amazon MWAA

- MWAA is managed Airflow — you cannot SSH into workers, schedulers, or web servers. No access to airflow.cfg directly. Configuration is via environment variables with the `AIRFLOW__` prefix or through the MWAA console/API.
- DAGs must be in S3 — MWAA syncs DAGs from an S3 bucket. DAG parse errors happen during sync, not at deploy time. Check the DAGProcessing log group for import errors.
- requirements.txt runs at environment startup — if a package fails to install, the entire environment can become unhealthy. Always test requirements locally first. Version pins are critical.
- plugins.zip is extracted to the Airflow plugins directory — custom operators, hooks, and sensors go here. The ZIP structure must match Airflow's expected plugin layout.
- VPC requires private subnets with NAT gateway — MWAA environments need private subnets with outbound internet access (via NAT) for PyPI package installation and AWS API calls. No public subnet support.
- Web server access modes: PUBLIC_ONLY or PRIVATE_ONLY — public mode creates an internet-facing endpoint; private mode requires VPC endpoint access. Switching modes requires environment update.
- Execution role is the single IAM identity for all Airflow operations — S3 DAG access, CloudWatch logging, Secrets Manager, KMS, and any AWS services called from DAGs all use this role.
- Airflow version upgrades are in-place but require careful testing — not all DAGs are compatible across Airflow versions. Test DAGs against the target version before upgrading.
- Worker auto-scaling is based on task queue depth — min/max workers are configurable. Scaling takes minutes, not seconds. Tasks may queue during scale-up.
- Secrets backend (Secrets Manager or Parameter Store) requires specific naming conventions — connection IDs and variables must follow the configured prefix pattern.
- CloudWatch log groups are per-component — separate log groups for Scheduler, Worker, WebServer, DAGProcessing, and Task logs. Each can be enabled/disabled independently.
- Environment updates can take 20-30 minutes — updates are not instant. The environment enters UPDATING status and may be partially unavailable during the update.

## Anti-hallucination rules

1. Always cite specific environment configurations, IAM policies, log entries, or API responses as evidence. Never guess at environment settings.
2. MWAA does NOT allow SSH access to any component. Never suggest connecting to workers or schedulers directly.
3. DAGs must be in S3 — never suggest placing DAGs on local file systems or using git-sync directly.
4. requirements.txt failures can break the entire environment. Never dismiss package installation errors as non-critical.
5. VPC must use private subnets with NAT. Never suggest public subnets for MWAA.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Environment | A1-A3 | Creation failures, update failures, version upgrades |
| B — DAGs | B1-B3 | DAG import errors, task failures, scheduling issues |
| C — Connectivity | C1-C2 | VPC networking, web server access |
| D — Performance | D1-D2 | Worker scaling, task queue backlog |
| E — Plugins | E1-E2 | Custom plugins, requirements.txt |
| F — Logging | F1-F2 | CloudWatch logs, task logs |
| G — Security | G1-G2 | IAM execution role, secrets backend |
| H — Integration | H1-H2 | S3 DAG bucket, connections |
| Z — Catch-All | Z1 | General troubleshooting |
