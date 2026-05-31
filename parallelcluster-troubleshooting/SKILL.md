---
name: parallelcluster-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS ParallelCluster problems
  by analyzing cluster creation, compute fleet, Slurm scheduler, shared storage,
  networking, job execution, auto-scaling, and security configurations using
  structured runbooks.
  Activate when: cluster creation failures, update issues, deletion problems,
  compute fleet errors, Slurm scheduler issues, shared storage (EFS/FSx)
  problems, scratch storage failures, VPC configuration issues, multi-AZ
  problems, job submission errors, job failures, auto-scaling issues,
  capacity problems, IAM permission errors, SSH access issues, or the user
  says something is wrong with ParallelCluster without naming specific symptoms.
compatibility: >
  Requires AWS CLI, pcluster CLI, and SDK access with EC2, CloudFormation,
  EFS, FSx, S3, CloudWatch, IAM, and VPC permissions.
---

# ParallelCluster Diagnostics

## When to use

Any AWS ParallelCluster investigation where the console alone is insufficient — cluster lifecycle management, compute fleet troubleshooting, Slurm scheduler debugging, shared storage issues, networking configuration, job execution problems, scaling behavior, or security configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
pcluster list-clusters
pcluster describe-cluster --cluster-name <cluster-name>
pcluster get-cluster-stack-events --cluster-name <cluster-name>
pcluster describe-compute-fleet --cluster-name <cluster-name>
pcluster list-cluster-log-streams --cluster-name <cluster-name>
```

### Step 2 — Domain deep dive

```
pcluster get-cluster-log-events --cluster-name <cluster-name> --log-stream-name <stream>
aws cloudformation describe-stack-events --stack-name <stack-name>
aws ec2 describe-instances --filters "Name=tag:parallelcluster:cluster-name,Values=<cluster-name>"
aws efs describe-file-systems --query 'FileSystems[?Tags[?Key==`parallelcluster:cluster-name` && Value==`<cluster-name>`]]'
aws fsx describe-file-systems
```

### Step 3 — Detailed investigation

```
pcluster ssh --cluster-name <cluster-name>
# On head node:
sinfo
squeue
scontrol show nodes
scontrol show partitions
cat /var/log/parallelcluster/clustermgtd
cat /var/log/parallelcluster/slurm_resume.log
cat /var/log/parallelcluster/slurm_suspend.log
```

Read `references/guardrails.md` before concluding on any ParallelCluster issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `pcluster describe-cluster` | Cluster status and config |
| `pcluster get-cluster-stack-events` | CloudFormation events |
| `pcluster describe-compute-fleet` | Compute fleet status |
| `pcluster list-cluster-log-streams` | Available log streams |
| `pcluster get-cluster-log-events` | Log content |
| `sinfo` | Slurm node/partition status |
| `squeue` | Job queue status |
| `scontrol show nodes` | Detailed node info |
| `sacct` | Job accounting data |
| `cloudformation describe-stack-events` | Stack events |

## Gotchas: ParallelCluster

- Cluster creation uses CloudFormation under the hood. Stack failures are the primary source of creation errors.
- Head node is a single EC2 instance — if it fails, the cluster is unavailable. Use placement groups and EBS snapshots for resilience.
- Slurm is the only supported scheduler in ParallelCluster 3.x. SGE and Torque were removed.
- Compute fleet uses dynamic nodes that are launched on demand by Slurm's power-saving plugin. Nodes are terminated after idle timeout.
- Shared storage options: EFS (elastic, multi-AZ), FSx for Lustre (high-performance), and EBS (head node only by default).
- FSx for Lustre scratch filesystems are temporary — data is lost when the cluster is deleted unless backed up to S3.
- Multi-queue support allows different instance types per Slurm partition. Each queue maps to a Slurm partition.
- Custom AMIs must be built with pcluster build-image. Using non-ParallelCluster AMIs causes failures.
- The head node needs internet access (or VPC endpoints) for package installation and AWS API calls.
- Auto-scaling is managed by clustermgtd and Slurm's power-saving plugin, not EC2 Auto Scaling groups.

## Anti-hallucination rules

1. Always check CloudFormation stack events for cluster creation/update failures — they contain the actual error.
2. Never claim SGE or Torque are supported — ParallelCluster 3.x only supports Slurm.
3. FSx for Lustre scratch filesystems are NOT persistent — data is lost on cluster deletion.
4. Compute nodes are dynamic — they are launched by Slurm, not pre-provisioned.
5. Custom AMIs must be built with pcluster build-image — arbitrary AMIs will not work.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A3 | Creation failures, update, deletion |
| B — Compute | B1-B2 | Compute fleet, Slurm scheduler |
| C — Storage | C1-C2 | Shared storage (EFS/FSx), scratch |
| D — Networking | D1-D2 | VPC config, multi-AZ |
| E — Jobs | E1-E2 | Job submission, job failures |
| F — Scaling | F1-F2 | Auto-scaling, capacity |
| G — Security | G1-G2 | IAM, SSH access |
| Z — Catch-All | Z1 | General troubleshooting |
