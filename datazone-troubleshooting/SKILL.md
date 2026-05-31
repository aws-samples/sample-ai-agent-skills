---
name: datazone-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon DataZone problems
  by analyzing domain creation, projects, data assets, subscriptions,
  governance, business catalog, and data quality using structured runbooks.
  Activate when: domain creation failures, project issues, data asset
  publishing errors, subscription problems, governance policy failures,
  catalog search issues, data quality errors, or the user says something
  is wrong with DataZone without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with DataZone, Glue, S3, Redshift,
  Athena, IAM, CloudWatch, CloudTrail permissions.
---

# DataZone Diagnostics

## When to use

Any Amazon DataZone investigation — domain creation, projects, data assets, subscriptions, governance, business catalog, or data quality.

## Investigation workflow

### Step 1 — Collect and triage

```
aws datazone list-domains --max-results 20
aws datazone get-domain --identifier <domain-id>
aws datazone list-projects --domain-identifier <domain-id> --max-results 20
```

### Step 2 — Domain deep dive

```
aws datazone get-project --domain-identifier <domain-id> --identifier <project-id>
aws datazone list-environments --domain-identifier <domain-id> --project-identifier <project-id>
aws datazone search --domain-identifier <domain-id> --search-scope ASSET --search-text "*"
aws datazone list-subscriptions --domain-identifier <domain-id> --max-results 20
```

### Step 3 — Detailed investigation

```
aws datazone get-asset --domain-identifier <domain-id> --identifier <asset-id>
aws datazone get-subscription --domain-identifier <domain-id> --identifier <sub-id>
aws datazone get-environment --domain-identifier <domain-id> --identifier <env-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=datazone.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any DataZone issue.

## Gotchas: DataZone

- DataZone is a data governance and catalog service, not a data processing service.
- Domains are the top-level organizational unit containing projects and assets.
- Projects contain environments where data producers and consumers work.
- Data assets are metadata entries in the catalog — they reference data, not store it.
- Subscriptions grant access to data assets between projects.
- Environments connect to data sources (Glue, Redshift, Athena) for actual data access.
- Business glossary provides standardized terminology across the organization.

## Anti-hallucination rules

1. DataZone manages metadata and governance — it doesn't store or process data.
2. Domains must be created before any other DataZone resources.
3. Subscriptions require approval workflow — they're not instant access grants.
4. Environments connect to actual data sources — DataZone doesn't replace them.
5. Data quality rules are defined in DataZone but executed by underlying services.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Domain | A1-A2 | Domain creation, configuration |
| B — Projects | B1-B2 | Project management, environments |
| C — Assets | C1-C2 | Data assets, publishing |
| D — Subscriptions | D1-D2 | Subscription requests, fulfillment |
| E — Governance | E1-E2 | Policies, approval workflows |
| F — Catalog | F1 | Business catalog, glossary |
| G — Quality | G1-G2 | Data quality, profiling |
| Z — Catch-All | Z1 | General troubleshooting |
