# AWS IoT Analytics Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT Analytics problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT Analytics — channels, pipelines, data stores, datasets, SQL queries, container activities, and data retention.

### Activate When

- Channel data ingestion failures
- Pipeline processing or activity errors
- Data store query or storage issues
- Dataset computation failures
- SQL query errors in datasets
- Container activity failures
- Data retention or cleanup issues
- General IoT Analytics errors

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Channels** | A1–A2 | Ingestion failures, channel configuration |
| **B — Pipelines** | B1–B3 | Processing errors, activity failures, transforms |
| **C — Data Stores** | C1–C2 | Storage issues, partitioning |
| **D — Datasets** | D1–D3 | SQL queries, container activities, scheduling |
| **E — Retention** | E1–E2 | Data retention, cleanup |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with IoTAnalytics permissions
- Permissions: `iotanalytics:Describe*`, `iotanalytics:List*`, `iotanalytics:Get*`, `iotanalytics:SampleChannelData`

---

## License

MIT-0
