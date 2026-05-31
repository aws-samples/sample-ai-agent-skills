---
name: emr-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon EMR problems by
  analyzing cluster launch failures, termination issues, bootstrap actions,
  step failures, Spark tuning, YARN resource management, shuffle issues,
  EMRFS/S3 issues, HDFS problems, VPC configuration, security groups,
  managed scaling, instance fleets, EMR on EKS virtual clusters, job run
  failures, EMR Serverless application failures, capacity issues, Kerberos
  authentication, Lake Formation integration, EMR Studio, Jupyter issues,
  and following structured runbooks. Activate when: cluster won't launch,
  unexpected termination, bootstrap action failures, step execution errors,
  Spark performance issues, YARN resource contention, shuffle failures,
  S3 read/write errors, HDFS capacity issues, VPC connectivity problems,
  scaling not working, EMR on EKS failures, Serverless job errors, security
  configuration issues, or the user says something is wrong with EMR.
compatibility: >
  Requires AWS CLI or SDK access with EMR, EC2, S3, IAM, CloudWatch,
  CloudTrail, and optionally EKS, Lake Formation, and VPC permissions.
  Some operations require EMR cluster access via SSH or EMR Studio.
---

# Amazon EMR Diagnostics

## When to use

Any Amazon EMR investigation where the console alone is insufficient — cluster lifecycle issues, application performance tuning, storage problems, networking configuration, scaling behavior, EMR on EKS, EMR Serverless, security configuration, or notebook/studio access.

## Investigation workflow

### Step 1 — Collect and triage

```
aws emr describe-cluster --cluster-id <cluster-id> --query 'Cluster.{State:Status.State,StateChangeReason:Status.StateChangeReason,Timeline:Status.Timeline}'
aws emr list-clusters --active --query 'Clusters[*].{Id:Id,Name:Name,State:Status.State}'
version: "1.0.0"
last_updated: "2025-04-12"
aws emr list-steps --cluster-id <cluster-id> --query 'Steps[*].{Id:Id,Name:Name,State:Status.State}'
version: "1.0.0"
last_updated: "2025-04-12"
aws emr list-bootstrap-actions --cluster-id <cluster-id>
```

### Step 2 — Domain deep dive

```
aws emr describe-cluster --cluster-id <cluster-id> --query 'Cluster.{Ec2:Ec2InstanceAttributes,Apps:Applications,Config:Configurations}'
aws emr list-instance-groups --cluster-id <cluster-id> --query 'InstanceGroups[*].{Type:InstanceGroupType,InstanceType:InstanceType,Running:RunningInstanceCount,Requested:RequestedInstanceCount}'
aws emr describe-step --cluster-id <cluster-id> --step-id <step-id>
aws cloudwatch get-metric-statistics --namespace AWS/ElasticMapReduce --metric-name IsIdle --dimensions Name=JobFlowId,Value=<cluster-id> --start-time <start> --end-time <end> --period 300 --statistics Average
```

### Step 3 — Detailed investigation

```
aws s3 ls s3://<log-bucket>/<cluster-id>/
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticmapreduce.amazonaws.com --max-results 20
aws emr describe-cluster --cluster-id <cluster-id> --query 'Cluster.{SecurityConfig:SecurityConfiguration,ServiceRole:ServiceRole,AutoScaling:AutoScalingRole}'
```

Read `references/emr-guardrails.md` before concluding on any EMR issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `emr describe-cluster` | Check cluster status, configuration, and state change reason |
| `emr list-steps` | Check step execution status |
| `emr list-bootstrap-actions` | Check bootstrap action configuration |
| `emr list-instance-groups` | Check instance group status and capacity |
| `emr describe-step` | Get detailed step execution information |
| `emr list-clusters` | List active/terminated clusters |
| S3 log bucket | Access cluster logs, step logs, application logs |
| CloudWatch metrics | Monitor cluster performance metrics |
| YARN ResourceManager UI | Check application status and resource allocation |

## Gotchas: Amazon EMR

- EMR cluster logs are in S3, not CloudWatch by default. Logs are written to the configured S3 log bucket. Check s3://<log-bucket>/<cluster-id>/ for step logs, application logs, and node logs. Enable CloudWatch logging separately if needed.
- Bootstrap actions run BEFORE applications start. If a bootstrap action fails, the cluster terminates. Bootstrap actions have a timeout. They run on all nodes (master, core, task). Debug by checking bootstrap action logs in S3.
- YARN resource allocation determines application performance. Spark executors, memory, and cores are controlled by YARN. Misconfigured YARN settings lead to underutilization or OOM errors. Check yarn-site.xml and spark-defaults.conf.
- EMRFS consistent view is deprecated. Use S3 strong consistency (default since Dec 2020). Do not configure EMRFS consistent view for new clusters. S3 provides strong read-after-write consistency natively.
- Managed scaling adjusts capacity based on workload. It adds/removes instances automatically. Scaling has a cooldown period. Minimum and maximum units must be configured. Scaling decisions are based on YARN metrics.
- EMR on EKS runs Spark on existing EKS clusters. It requires a virtual cluster, execution role, and job template. Debugging requires checking both EMR and EKS logs. Pod scheduling issues are common.
- EMR Serverless automatically provisions resources. No cluster management needed. Applications have pre-initialized capacity settings. Job failures often relate to IAM permissions or resource limits.
- Security configurations are immutable after cluster creation. Kerberos, encryption, and Lake Formation settings cannot be changed on a running cluster. Create a new cluster with the correct security configuration.
- Instance fleets provide flexibility in instance selection. EMR chooses from a list of instance types based on availability and price. On-Demand and Spot instances can be mixed. Spot interruptions can affect running tasks.

## Anti-hallucination rules

1. Always cite specific cluster IDs, step IDs, or log entries as evidence.
2. EMRFS consistent view is deprecated. Never recommend configuring it.
3. Bootstrap actions run before applications. Never suggest modifying running applications via bootstrap.
4. Security configurations are immutable. Never suggest changing security config on a running cluster.
5. S3 has strong consistency natively. Never suggest EMRFS workarounds for consistency.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 28 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Cluster | A1-A4 | Launch failures, termination, bootstrap actions, step failures |
| B — Performance | B1-B3 | Spark tuning, YARN resource management, shuffle issues |
| C — Storage | C1-C2 | EMRFS/S3 issues, HDFS |
| D — Networking | D1-D2 | VPC config, security groups |
| E — Scaling | E1-E2 | Managed scaling, instance fleet |
| F — EMR on EKS | F1-F2 | Virtual cluster, job run failures |
| G — EMR Serverless | G1-G2 | Application failures, capacity |
| H — Security | H1-H2 | Kerberos, Lake Formation integration |
| I — Notebooks | I1-I2 | EMR Studio, Jupyter issues |
| Z — Catch-All | Z1 | General troubleshooting |
