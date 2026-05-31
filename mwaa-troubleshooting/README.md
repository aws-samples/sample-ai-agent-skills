# Amazon MWAA Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Managed Workflows for Apache Airflow (MWAA) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon MWAA when the console alone isn't enough — environment health analysis, DAG import debugging, task failure investigation, scheduling issues, VPC networking, web server access, worker scaling, plugin/requirements problems, logging issues, IAM permissions, secrets backend, and S3 bucket sync.

### Activate When

- Environment creation or update failures
- Airflow version upgrade issues
- DAG import/parse errors
- Task execution failures
- DAG scheduling not triggering
- VPC networking connectivity problems
- Web server inaccessible (public or private mode)
- Worker auto-scaling not working
- Task queue backlog growing
- Custom plugin loading failures
- requirements.txt package installation errors
- CloudWatch log delivery issues
- Task logs missing or incomplete
- IAM execution role permission errors
- Secrets backend configuration problems
- S3 DAG bucket sync failures
- Airflow connection configuration issues

---

## Skill Structure

```
mwaa-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-update-failures.md
    ├── A3-version-upgrades.md
    ├── B1-dag-import-errors.md
    ├── B2-task-failures.md
    ├── B3-scheduling-issues.md
    ├── C1-vpc-networking.md
    ├── C2-web-server-access.md
    ├── D1-worker-scaling.md
    ├── D2-task-queue-backlog.md
    ├── E1-custom-plugins.md
    ├── E2-requirements-txt.md
    ├── F1-cloudwatch-logs.md
    ├── F2-task-logs.md
    ├── G1-iam-execution-role.md
    ├── G2-secrets-backend.md
    ├── H1-s3-dag-bucket.md
    ├── H2-connections.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Environment** | A1–A3 | Creation failures, update failures, version upgrades |
| **B — DAGs** | B1–B3 | DAG import errors, task failures, scheduling issues |
| **C — Connectivity** | C1–C2 | VPC networking, web server access |
| **D — Performance** | D1–D2 | Worker scaling, task queue backlog |
| **E — Plugins** | E1–E2 | Custom plugins, requirements.txt |
| **F — Logging** | F1–F2 | CloudWatch logs, task logs |
| **G — Security** | G1–G2 | IAM execution role, secrets backend |
| **H — Integration** | H1–H2 | S3 DAG bucket, connections |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering managed service boundaries, VPC private subnet requirements, execution role scope, DAG S3 sync behavior, requirements.txt risks, plugin ZIP structure, web server access modes, worker scaling characteristics, secrets backend naming, CloudWatch log group structure, environment update duration, and Airflow version upgrade compatibility.

---

## Investigation Workflow

1. **Triage** — List environments, describe environment status, check CloudWatch metrics and log groups
2. **Deep Dive** — Examine S3 DAG/plugin/requirements paths, execution role, Airflow CLI via token
3. **Detailed** — Review CloudTrail events, filter error logs, check VPC/NAT/security groups

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `airflow:*`, `s3:GetObject`, `s3:ListBucket`, `iam:GetRole`, `iam:ListAttachedRolePolicies`, `logs:FilterLogEvents`, `logs:DescribeLogGroups`, `cloudtrail:LookupEvents`, `ec2:DescribeSecurityGroups`, `ec2:DescribeSubnets`, `ec2:DescribeNatGateways`, `secretsmanager:ListSecrets`
- Airflow CLI access via `aws mwaa create-cli-token`

---

## Usage Examples

```
# List all MWAA environments
aws mwaa list-environments

# Get environment details
aws mwaa get-environment --name my-airflow-env

# Get CLI token for Airflow commands
aws mwaa create-cli-token --name my-airflow-env

# Check DAG processing errors
aws logs filter-log-events --log-group-name airflow-my-airflow-env-DAGProcessing --filter-pattern "ERROR"

# List DAGs in S3
aws s3 ls s3://my-airflow-bucket/dags/

# Check execution role
aws iam get-role --role-name my-mwaa-execution-role
```

---

## License

MIT-0
