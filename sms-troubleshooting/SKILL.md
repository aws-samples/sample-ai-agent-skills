---
name: sms-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Server Migration Service
  (SMS) problems by analyzing replication jobs, server catalog, AMI generation,
  application migration, multi-server migration, and scheduling using
  structured runbooks.
  Activate when: replication job failures, server catalog issues, AMI generation
  errors, application migration problems, multi-server coordination issues,
  scheduling errors, or the user says something is wrong with SMS without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SMS, EC2, S3, IAM, CloudWatch,
  CloudTrail permissions. SMS Connector deployed in VMware environment.
---

# SMS Diagnostics

## When to use

Any AWS Server Migration Service investigation — replication jobs, server catalog, AMI generation, application migration, multi-server coordination, or scheduling.

## Investigation workflow

### Step 1 — Collect and triage

```
aws sms get-connectors --max-results 20
aws sms get-servers --max-results 20
aws sms get-replication-jobs --max-results 20
```

### Step 2 — Domain deep dive

```
aws sms get-replication-runs --replication-job-id <job-id> --max-results 10
aws sms list-apps --max-results 20
aws sms get-app --app-id <app-id>
```

### Step 3 — Detailed investigation

```
aws sms get-app-replication-configuration --app-id <app-id>
aws sms get-app-launch-configuration --app-id <app-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sms.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any SMS issue.

## Gotchas: SMS

- SMS is being replaced by Application Migration Service (MGN) for new migrations.
- SMS Connector is required in the source VMware/Hyper-V environment.
- Replication creates incremental snapshots, not full copies each time.
- AMIs are generated from replication snapshots.
- Application migration groups multiple servers for coordinated migration.
- SMS has a 90-day replication limit per server.

## Anti-hallucination rules

1. SMS is legacy — recommend Application Migration Service (MGN) for new migrations.
2. SMS Connector must be deployed and healthy before replication can start.
3. Replication jobs have a 90-day limit — they must be completed within this window.
4. AMI generation happens after replication run completes, not during.
5. Application migration coordinates multiple servers but doesn't guarantee atomic migration.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Replication | A1-A2 | Replication jobs, run failures |
| B — Catalog | B1-B2 | Server catalog, connector |
| C — AMI | C1-C2 | AMI generation, launch config |
| D — Application | D1-D2 | App migration, server groups |
| E — Multi-Server | E1 | Multi-server coordination |
| F — Scheduling | F1 | Replication scheduling |
| Z — Catch-All | Z1 | General troubleshooting |
