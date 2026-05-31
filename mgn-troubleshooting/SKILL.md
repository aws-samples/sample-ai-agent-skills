---
name: mgn-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Application Migration
  Service (MGN) problems by analyzing source server registration, replication
  status, launch templates, cutover operations, post-migration validation,
  agent installation, and following structured runbooks.
  Activate when: source server not replicating, replication lag, launch
  failures, cutover errors, post-migration issues, agent problems, or the
  user says something is wrong with MGN without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with mgn, ec2, iam, cloudtrail, and
  ssm permissions. MGN requires source servers with the AWS Replication
  Agent installed.
---

# AWS Application Migration Service Diagnostics

## When to use

Any AWS MGN investigation where the console alone is insufficient — source server registration, replication issues, launch template errors, cutover failures, post-migration validation, or agent installation problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mgn describe-source-servers --query 'items[*].{Id:sourceServerID,Hostname:sourceProperties.identificationHints.hostname,State:lifeCycle.state,Replication:dataReplicationInfo.dataReplicationState}'
version: "1.0.0"
last_updated: "2025-04-12"
aws mgn describe-replication-configuration-templates
aws mgn describe-launch-configuration-templates
```

### Step 2 — Domain deep dive

```
aws mgn describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].dataReplicationInfo'
aws mgn get-launch-configuration --source-server-id <id>
aws mgn get-replication-configuration --source-server-id <id>
aws mgn describe-jobs --filters 'fromDate=2024-01-01T00:00:00Z'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mgn.amazonaws.com --max-results 20
aws mgn describe-job-log-items --job-id <job-id>
aws mgn list-waves
aws mgn list-applications
```

Read `references/guardrails.md` before concluding on any MGN issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `mgn describe-source-servers` | Check source server status |
| `mgn get-replication-configuration` | Review replication settings |
| `mgn get-launch-configuration` | Review launch settings |
| `mgn describe-jobs` | List migration jobs |
| `mgn start-cutover` | Initiate cutover |
| `mgn finalize-cutover` | Finalize migration |
| `mgn list-waves` | List migration waves |
| `mgn list-applications` | List migration applications |

## Gotchas: AWS Application Migration Service

- MGN and DRS share the same replication agent. A source server can only be registered with ONE service at a time (MGN or DRS, not both).
- Cutover is a multi-step process: StartCutover → validate → FinalizeCutover. Finalizing cutover stops replication permanently. This is irreversible.
- Waves and Applications are organizational constructs for grouping servers. They don't affect replication or launch behavior directly.
- Post-launch actions use SSM Run Command. The recovery instance must have SSM agent and connectivity to SSM endpoints.
- MGN is being superseded by DRS for some use cases. Check current AWS recommendations for new migrations.
- Launch template changes don't retroactively apply to existing source servers.

## Anti-hallucination rules

1. Always cite specific source server IDs, replication states, or API responses as evidence.
2. MGN agent is the same as DRS agent. Never suggest installing both.
3. FinalizeCutover is irreversible — it stops replication permanently. Never omit this warning.
4. Waves/Applications are organizational only. Never claim they affect replication.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Source Server | A1-A2 | Registration, source properties |
| B — Replication | B1-B3 | Replication lag, stalled replication, initial sync |
| C — Launch Template | C1-C2 | Launch errors, instance mapping |
| D — Cutover | D1-D3 | Cutover launch, cutover finalization, rollback |
| E — Post-Migration | E1-E2 | Validation, application testing |
| F — Agent | F1-F2 | Agent installation, agent connectivity |
| Z — Catch-All | Z1 | General troubleshooting |
