---
name: kendra-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Kendra problems
  by analyzing index creation, data source sync, query relevance, FAQ
  management, custom document enrichment, access control, experience
  builder, and following structured runbooks. Activate when: index
  creation failures, data source sync errors, poor query relevance,
  FAQ issues, document enrichment failures, access control problems,
  experience builder errors, or the user says something is wrong with
  Kendra without naming specific symptoms.
compatibility: >
  Requires AWS CLI with Kendra, S3, IAM, CloudWatch, Lambda, and
  optionally Secrets Manager permissions.
---

# Amazon Kendra Diagnostics

## When to use

Any Amazon Kendra investigation — index creation, data source sync, query relevance, FAQ, custom document enrichment, access control, or experience builder.

## Investigation workflow

### Step 1 — Collect and triage

```
aws kendra list-indices
aws kendra describe-index --id <index-id>
aws kendra list-data-sources --index-id <index-id>
aws kendra list-faqs --index-id <index-id>
```

### Step 2 — Domain deep dive

```
aws kendra describe-data-source --id <ds-id> --index-id <index-id>
aws kendra list-data-source-sync-jobs --id <ds-id> --index-id <index-id>
aws kendra describe-experience --id <exp-id> --index-id <index-id>
aws kendra list-entity-personas --id <exp-id> --index-id <index-id>
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/Kendra --metric-name IndexQueryCount --dimensions Name=IndexId,Value=<index-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws kendra describe-query-suggestions-config --index-id <index-id>
aws kendra list-thesauri --index-id <index-id>
```

Read `references/guardrails.md` before concluding on any Kendra issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `kendra list-indices` | List indices and status |
| `kendra describe-index` | Get index configuration |
| `kendra list-data-sources` | List data sources |
| `kendra list-data-source-sync-jobs` | Check sync history |
| `kendra list-faqs` | List FAQ resources |
| `kendra query` | Test search queries |
| CloudWatch metrics | Monitor query volume and latency |

## Gotchas: Amazon Kendra

- Index creation takes 15-30 minutes. The index is not queryable until ACTIVE.
- Data source sync is asynchronous. Documents are not searchable until sync completes.
- Query relevance depends on document quality and metadata. Poor results often stem from missing metadata.
- FAQs use specific CSV/JSON format. Format errors cause import failures.
- Custom document enrichment runs Lambda during ingestion. Lambda errors block document processing.
- Access control uses ACLs from data sources. Misconfigured ACLs cause missing results.
- Experience builder requires IAM Identity Center. SSO must be configured first.

## Anti-hallucination rules

1. Always cite index IDs, data source IDs, or API responses as evidence.
2. Index creation is not instant. Never assume immediate availability.
3. Sync must complete before documents are searchable.
4. Query relevance tuning requires experimentation. No magic configuration.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Index | A1-A2 | Index creation, index configuration |
| B — Data Sources | B1-B3 | Sync failures, connector config, crawl issues |
| C — Query | C1-C3 | Relevance, suggestions, filtering |
| D — FAQ | D1-D2 | FAQ import, FAQ quality |
| E — Enrichment | E1-E2 | Custom enrichment, Lambda errors |
| F — Access | F1-F2 | ACL configuration, token-based access |
| G — Experience | G1-G2 | Experience builder, SSO integration |
| Z — Catch-All | Z1 | General troubleshooting |
