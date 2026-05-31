# ParallelCluster Diagnostics Skill

Agent skill for investigating and troubleshooting AWS ParallelCluster problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS ParallelCluster when the console alone isn't enough — cluster lifecycle management, compute fleet issues, Slurm scheduler debugging, shared storage (EFS/FSx), networking, job execution, auto-scaling, and security configuration.

### Activate When

- Cluster creation failures
- Cluster update or deletion problems
- Compute fleet not launching or scaling
- Slurm scheduler errors or misconfigurations
- Shared storage (EFS/FSx) mount or performance issues
- Scratch filesystem problems
- VPC or subnet configuration issues
- Multi-AZ deployment problems
- Job submission failures
- Job execution errors or timeouts
- Auto-scaling not working as expected
- Capacity exhaustion
- IAM permission errors
- SSH access problems

---

## Skill Structure

```
parallelcluster-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-creation-failures.md
    ├── A2-update.md
    ├── A3-deletion.md
    ├── B1-compute-fleet.md
    ├── B2-slurm-scheduler.md
    ├── C1-shared-storage.md
    ├── C2-scratch.md
    ├── D1-vpc-config.md
    ├── D2-multi-az.md
    ├── E1-job-submission.md
    ├── E2-job-failures.md
    ├── F1-auto-scaling.md
    ├── F2-capacity.md
    ├── G1-iam.md
    ├── G2-ssh-access.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster** | A1–A3 | Creation failures, update, deletion |
| **B — Compute** | B1–B2 | Compute fleet, Slurm scheduler |
| **C — Storage** | C1–C2 | Shared storage (EFS/FSx), scratch |
| **D — Networking** | D1–D2 | VPC config, multi-AZ |
| **E — Jobs** | E1–E2 | Job submission, job failures |
| **F — Scaling** | F1–F2 | Auto-scaling, capacity |
| **G — Security** | G1–G2 | IAM, SSH access |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering CloudFormation stack errors, Slurm-only scheduler, FSx scratch persistence, dynamic compute nodes, custom AMI requirements, head node single point of failure, shared storage mount points, multi-queue configuration, networking requirements, job accounting, scaling cooldowns, and cluster configuration immutability.

---

## Investigation Workflow

1. **Triage** — Check cluster status, CloudFormation events, compute fleet state
2. **Deep Dive** — Examine logs, Slurm state, storage mounts, network configuration
3. **Detailed** — SSH to head node, inspect Slurm logs, check job output

---

## Prerequisites

- AWS CLI v2 and pcluster CLI v3 installed
- Permissions: `ec2:*`, `cloudformation:*`, `efs:*`, `fsx:*`, `s3:*`, `iam:*`, `cloudwatch:*`
- SSH key pair for head node access
- VPC with appropriate subnet configuration

---

## Usage Examples

```
# List clusters
pcluster list-clusters

# Check cluster status
pcluster describe-cluster --cluster-name my-hpc-cluster

# View stack events for creation failures
pcluster get-cluster-stack-events --cluster-name my-hpc-cluster

# Check compute fleet
pcluster describe-compute-fleet --cluster-name my-hpc-cluster

# SSH to head node
pcluster ssh --cluster-name my-hpc-cluster
```

---

## License

MIT-0
