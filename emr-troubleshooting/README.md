# Amazon EMR Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon EMR problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon EMR when the console alone isn't enough — cluster lifecycle management, application performance tuning, storage configuration, networking, scaling, EMR on EKS, EMR Serverless, security, and notebook/studio access.

### Activate When

- Cluster launch failures or unexpected termination
- Bootstrap action failures
- Step execution errors
- Spark performance issues or OOM errors
- YARN resource contention
- Shuffle failures or disk space issues
- EMRFS/S3 read/write errors
- HDFS capacity or replication issues
- VPC connectivity problems
- Security group configuration issues
- Managed scaling not working
- Instance fleet provisioning problems
- EMR on EKS virtual cluster or job failures
- EMR Serverless application or capacity issues
- Kerberos authentication errors
- Lake Formation integration problems
- EMR Studio access issues
- Jupyter notebook connectivity problems
- General EMR errors without clear symptoms

---

## Skill Structure

```
emr-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── A2-termination.md
    ├── A3-bootstrap-actions.md
    ├── A4-step-failures.md
    ├── B1-spark-tuning.md
    ├── B2-yarn-resource-management.md
    ├── B3-shuffle-issues.md
    ├── C1-emrfs-s3-issues.md
    ├── C2-hdfs.md
    ├── D1-vpc-config.md
    ├── D2-security-groups.md
    ├── E1-managed-scaling.md
    ├── E2-instance-fleet.md
    ├── F1-virtual-cluster.md
    ├── F2-job-run-failures.md
    ├── G1-serverless-application-failures.md
    ├── G2-serverless-capacity.md
    ├── H1-kerberos.md
    ├── H2-lake-formation-integration.md
    ├── I1-emr-studio.md
    ├── I2-jupyter-issues.md
    ├── Z1-general-troubleshooting.md
    ├── emr-guardrails.md
    └── emr-hallucination-patterns.yaml
```

---

## Runbook Library (28 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A4 | Launch failures, termination, bootstrap actions, step failures |
| **B — Performance** | B1–B3 | Spark tuning, YARN resource management, shuffle issues |
| **C — Storage** | C1–C2 | EMRFS/S3 issues, HDFS |
| **D — Networking** | D1–D2 | VPC config, security groups |
| **E — Scaling** | E1–E2 | Managed scaling, instance fleet |
| **F — EMR on EKS** | F1–F2 | Virtual cluster, job run failures |
| **G — EMR Serverless** | G1–G2 | Application failures, capacity |
| **H — Security** | H1–H2 | Kerberos, Lake Formation integration |
| **I — Notebooks** | I1–I2 | EMR Studio, Jupyter issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/emr-guardrails.md` covering S3 log location, bootstrap action lifecycle, YARN resource allocation, EMRFS consistency, managed scaling behavior, EMR on EKS architecture, EMR Serverless resource model, security configuration immutability, instance fleet selection, Spark memory model, cluster termination protection, and step concurrency.

---

## Investigation Workflow

1. **Triage** — Check cluster status, state change reason, step status, bootstrap actions
2. **Deep Dive** — Examine instance groups, configurations, CloudWatch metrics
3. **Detailed** — Review S3 logs, CloudTrail events, YARN/Spark application logs

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `elasticmapreduce:*`, `ec2:Describe*`, `s3:GetObject`, `s3:ListBucket`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Access to the EMR cluster S3 log bucket
- SSH access to cluster nodes for advanced debugging (optional)

---

## Usage Examples

```
# Check cluster status
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXX

# List active clusters
aws emr list-clusters --active

# Check step status
aws emr list-steps --cluster-id j-XXXXXXXXXXXXX

# Check instance groups
aws emr list-instance-groups --cluster-id j-XXXXXXXXXXXXX

# Access cluster logs
aws s3 ls s3://my-log-bucket/j-XXXXXXXXXXXXX/
```

---

## License

MIT-0
