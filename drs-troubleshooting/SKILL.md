---
name: drs-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elastic Disaster Recovery
  (DRS) problems by analyzing source server registration, replication status,
  launch configuration, recovery instance issues, failback operations, drill
  testing, networking configuration, and following structured runbooks.
  Activate when: source server not replicating, replication lag, launch
  failures, recovery instance errors, failback problems, drill test failures,
  networking connectivity issues, agent installation problems, or the user
  says something is wrong with DRS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with drs, ec2, iam, cloudtrail, and
  cloudwatch permissions. DRS requires source servers with the AWS
  Replication Agent installed.
---

# AWS Elastic Disaster Recovery Diagnostics

## When to use

Any AWS DRS investigation where the console alone is insufficient — source server registration failures, replication stalls or lag, launch configuration errors, recovery instance problems, failback issues, drill testing failures, or networking connectivity between source and target.

## Investigation workflow

### Step 1 — Collect and triage

```
aws drs describe-source-servers --query 'items[*].{Id:sourceServerID,Hostname:sourceProperties.identificationHints.hostname,State:lifeCycle.state,Lag:dataReplicationInfo.lagDuration}'
version: "1.0.0"
last_updated: "2025-04-12"
aws drs describe-replication-configuration-templates
aws drs describe-launch-configuration-templates
aws ec2 describe-instances --filters "Name=tag:aws:elasticdisasterrecovery:source-server-id,Values=*" --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name}'
```

### Step 2 — Domain deep dive

```
aws drs describe-source-servers --filters 'sourceServerIDs=<server-id>' --query 'items[0].dataReplicationInfo'
aws drs get-launch-configuration --source-server-id <server-id>
aws drs get-replication-configuration --source-server-id <server-id>
aws drs describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=drs.amazonaws.com --max-results 20
aws drs describe-job-log-items --job-id <job-id>
aws ec2 describe-security-groups --filters "Name=group-name,Values=*AWS Elastic Disaster Recovery*"
```

Read `references/guardrails.md` before concluding on any DRS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `drs describe-source-servers` | Check source server status and replication |
| `drs get-replication-configuration` | Review replication settings |
| `drs get-launch-configuration` | Review launch settings for recovery |
| `drs describe-jobs` | List recovery/drill/failback jobs |
| `drs describe-job-log-items` | Get detailed job logs |
| `drs start-recovery` | Initiate recovery launch |
| `drs start-failback-launch` | Start failback to source |
| `drs describe-recovery-instances` | Check recovery instance status |

## Gotchas: AWS Elastic Disaster Recovery

- The AWS Replication Agent must be installed on source servers BEFORE they appear in DRS. Agent installation requires outbound connectivity to DRS endpoints on TCP 443 and replication server on TCP 1500.
- Replication uses a staging area subnet in the target region. The staging area subnet must have outbound internet access or VPC endpoints for S3 and DRS APIs.
- Initial sync can take hours or days depending on disk size and bandwidth. Do NOT assume replication is broken during initial sync — check dataReplicationInfo.dataReplicationState.
- Launch configuration and replication configuration are PER source server. Template changes do NOT retroactively apply to existing source servers unless explicitly updated.
- Drill (test) instances are separate from actual recovery instances. Drill instances must be cleaned up manually or they incur ongoing costs.
- Failback requires the Failback Client to be installed on the recovery instance. Failback replicates data back to the original source location.
- DRS replication servers are EC2 instances in the staging area. They are automatically managed but appear in EC2 console. Do NOT terminate them manually.

## Anti-hallucination rules

1. Always cite specific source server IDs, replication states, or API responses as evidence.
2. DRS replication agent must be installed on source. Never assume auto-discovery.
3. Initial sync duration depends on data volume. Never claim replication is broken during initial sync without checking state.
4. Template changes don't auto-apply to existing servers. Always verify per-server config.
5. Drill instances incur costs until terminated. Never omit cleanup steps.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Source Server | A1-A3 | Registration, agent issues, source properties |
| B — Replication | B1-B3 | Replication lag, stalled replication, initial sync |
| C — Launch Config | C1-C2 | Launch template errors, instance type mapping |
| D — Recovery | D1-D3 | Recovery launch failures, recovery instance issues, post-launch |
| E — Failback | E1-E2 | Failback initiation, failback replication |
| F — Drill Testing | F1-F2 | Drill launch failures, drill cleanup |
| G — Networking | G1 | Staging area and connectivity |
| Z — Catch-All | Z1 | General troubleshooting |
