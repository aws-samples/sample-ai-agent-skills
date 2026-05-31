# AWS Batch Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Batch problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Batch when the console alone isn't enough — job failures, stuck RUNNABLE jobs, compute environment issues, job queue scheduling, job definition configuration, VPC networking, ECR access, array jobs, multi-node parallel, performance optimization, IAM roles, and secrets management.

### Activate When

- Job failures with non-zero exit codes
- Jobs stuck in RUNNABLE state
- Job timeout exceeded
- Compute environment creation or scaling failures
- EC2 vs Fargate compute issues
- Spot instance interruptions
- Job queue scheduling problems
- Job definition configuration errors
- Container configuration issues
- VPC networking problems
- ECR image pull failures
- Array job failures
- Multi-node parallel job issues
- Job throughput bottlenecks
- Resource optimization needed
- IAM role permission errors
- Secrets access failures
- General AWS Batch errors without clear symptoms

---

## Skill Structure

```
batch-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-job-failures.md
    ├── A2-stuck-runnable.md
    ├── A3-job-timeout.md
    ├── B1-compute-environment-failures.md
    ├── B2-ec2-vs-fargate.md
    ├── B3-spot-interruptions.md
    ├── C1-job-queue-issues.md
    ├── C2-scheduling.md
    ├── D1-job-definition-errors.md
    ├── D2-container-config.md
    ├── E1-vpc-config.md
    ├── E2-ecr-access.md
    ├── F1-array-jobs.md
    ├── F2-multi-node-parallel.md
    ├── G1-throughput.md
    ├── G2-resource-optimization.md
    ├── H1-iam-roles.md
    ├── H2-secrets.md
    ├── Z1-general-troubleshooting.md
    ├── batch-guardrails.md
    └── batch-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Jobs** | A1–A3 | Job failures, stuck RUNNABLE, timeout |
| **B — Compute** | B1–B3 | Compute environment failures, EC2 vs Fargate, spot interruptions |
| **C — Queue** | C1–C2 | Job queue issues, scheduling |
| **D — Definition** | D1–D2 | Job definition errors, container config |
| **E — Networking** | E1–E2 | VPC config, ECR access |
| **F — Array/Multi** | F1–F2 | Array jobs, multi-node parallel |
| **G — Performance** | G1–G2 | Throughput, resource optimization |
| **H — Security** | H1–H2 | IAM roles, secrets |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/batch-guardrails.md` covering RUNNABLE state meaning, EC2 vs Fargate capabilities, exit code interpretation, Spot interruption handling, array job behavior, multi-node parallel requirements, job queue priority, resource request matching, compute environment scaling, job retry strategies, fair-share scheduling, and managed vs unmanaged compute environments.

---

## Investigation Workflow

1. **Triage** — Check job status, compute environment state, job queue configuration
2. **Deep Dive** — Examine container logs, job definition, CE instance details
3. **Detailed** — Review CloudTrail events, ECS cluster, CloudWatch metrics

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `batch:Describe*`, `batch:List*`, `logs:GetLogEvents`, `ecs:DescribeClusters`, `ec2:DescribeInstances`, `cloudtrail:LookupEvents`
- CloudWatch Logs access for container logs (/aws/batch/job)
- Access to ECR for image troubleshooting

---

## Usage Examples

```
# Check job status and details
aws batch describe-jobs --jobs 12345678-1234-1234-1234-123456789012

# List failed jobs in a queue
aws batch list-jobs --job-queue my-queue --job-status FAILED --max-results 10

# Check compute environment
aws batch describe-compute-environments --compute-environments my-ce

# Get container logs
aws logs get-log-events --log-group-name /aws/batch/job --log-stream-name my-job-def/default/abc123

# Check job definition
aws batch describe-job-definitions --job-definition-name my-job-def --status ACTIVE
```

---

## License

MIT-0
