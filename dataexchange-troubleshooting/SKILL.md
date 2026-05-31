---
name: dataexchange-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Data Exchange
  problems by analyzing dataset creation, revision publishing,
  subscriptions, entitlements, S3 exports, API assets, Redshift
  query access, Lake Formation access, provider configuration,
  subscriber management, event notifications, and following
  structured runbooks.
  Activate when: dataset creation failures, revision publishing
  errors, subscription issues, entitlement problems, export
  failures, API asset errors, or the user says something is wrong
  with Data Exchange.
compatibility: >
  Requires AWS CLI or SDK access with dataexchange, s3, redshift,
  lakeformation, sns, iam, and cloudtrail permissions.
---

# AWS Data Exchange Diagnostics

## When to use

Any AWS Data Exchange investigation — dataset creation, revision publishing, subscriptions, entitlements, S3 exports, API assets, Redshift query access, Lake Formation access, provider configuration, subscriber management, or event notifications.

## Investigation workflow

### Step 1 — Collect and triage

```
aws dataexchange list-data-sets --query 'DataSets[*].{Name:Name,Id:Id,AssetType:AssetType,Origin:Origin}'
aws dataexchange list-event-actions --query 'EventActions[*].{Id:Id,Action:Action}'
aws dataexchange list-revisions --data-set-id <dataset-id> --query 'Revisions[*].{Id:Id,Finalized:Finalized,CreatedAt:CreatedAt}'
```

### Step 2 — Domain deep dive

```
aws dataexchange get-data-set --data-set-id <dataset-id>
aws dataexchange get-revision --data-set-id <dataset-id> --revision-id <revision-id>
aws dataexchange list-data-set-revisions --data-set-id <dataset-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dataexchange.amazonaws.com --max-results 20
aws dataexchange get-job --job-id <job-id>
aws dataexchange list-jobs --query 'Jobs[?State==`ERROR`]'
```

Read `references/guardrails.md` before concluding on any Data Exchange issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `dataexchange list-data-sets` | List data sets |
| `dataexchange get-data-set` | Get data set details |
| `dataexchange list-revisions` | List revisions for a data set |
| `dataexchange get-job` | Check export/import job status |
| `dataexchange list-jobs` | List all jobs with filters |
| `dataexchange create-revision` | Create new revision |
| `dataexchange send-api-asset` | Invoke API asset |

## Gotchas: AWS Data Exchange

- Data Exchange has TWO roles: Provider (publishes data) and Subscriber (consumes data). Each has different permissions and workflows.
- Revisions must be FINALIZED before they are visible to subscribers. Unfinalized revisions are drafts.
- S3 export jobs are asynchronous. Job status must be polled — there is no synchronous export.
- API assets proxy requests to the provider's API Gateway. Errors may originate from the provider's backend, not Data Exchange.
- Entitled data sets appear in the subscriber's account but the underlying data remains in the provider's account until exported.
- Lake Formation and Redshift data shares use AWS RAM (Resource Access Manager) for cross-account sharing.
- Event actions can automatically publish new revisions when S3 objects are created, but require correct S3 event configuration.

## Anti-hallucination rules

1. Always cite specific data set IDs, revision IDs, or job IDs as evidence.
2. Provider and subscriber roles have different API permissions. Never conflate them.
3. Entitled data is NOT automatically copied to the subscriber's account. Export is required.
4. API assets are proxied through Data Exchange. Backend errors are the provider's responsibility.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Data Sets | A1-A2 | Dataset creation, revision publishing |
| B — Subscriptions | B1-B2 | Subscription errors, entitlement issues |
| C — Exports | C1-C2 | S3 export failures, API asset errors |
| D — Data Access | D1-D2 | Redshift query access, Lake Formation access |
| E — Provider/Subscriber | E1-E2 | Provider configuration, subscriber management |
| F — Events | F1 | Event notifications |
| Z — Catch-All | Z1 | General troubleshooting |
