---
name: supplychain-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Supply Chain
  problems by analyzing instance creation, data integration, demand
  planning, supply planning, inventory visibility, connectors,
  and S3 integration using structured runbooks.
  Activate when: instance creation failures, data integration errors,
  demand planning issues, supply planning problems, inventory visibility
  failures, connector errors, S3 integration issues, or the user says
  something is wrong with Supply Chain.
compatibility: >
  Requires AWS CLI or SDK access with scn, s3, iam, kms,
  cloudtrail, and cloudwatch permissions.
---

# AWS Supply Chain Diagnostics

## When to use

Any AWS Supply Chain investigation — instance creation, data integration, demand planning, supply planning, inventory visibility, insights generation, connector configuration, or S3 integration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws scn list-instances --query 'instances[*].{Id:instanceId,Name:instanceName,State:state}'
aws scn get-instance --instance-id <instance-id>
aws scn list-data-integration-flows --instance-id <instance-id>
```

### Step 2 — Domain deep dive

```
aws scn get-data-integration-flow --instance-id <instance-id> --name <flow-name>
aws scn get-data-lake-dataset --instance-id <instance-id> --namespace <namespace> --name <dataset-name>
aws scn get-bill-of-materials-import-job --instance-id <instance-id> --job-id <job-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scn.amazonaws.com --max-results 20
aws scn list-data-lake-datasets --instance-id <instance-id> --namespace <namespace>
aws s3 ls s3://<supply-chain-bucket>/<prefix>/
```

Read `references/guardrails.md` before concluding on any Supply Chain issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `scn list-instances` | List Supply Chain instances |
| `scn get-instance` | Get instance details |
| `scn list-data-integration-flows` | List data integration flows |
| `scn get-data-integration-flow` | Get flow details |
| `scn list-data-lake-datasets` | List data lake datasets |
| `scn get-data-lake-dataset` | Get dataset details |
| `scn get-bill-of-materials-import-job` | Get BOM import job status |

## Gotchas: AWS Supply Chain

- Supply Chain instances are the top-level resource. All operations (data integration, planning, visibility) are scoped to an instance.
- Data integration flows connect external data sources to the Supply Chain data lake. Flows can fail silently — always check flow execution status.
- The data lake uses predefined namespaces and schemas. Custom data must conform to the expected schema or transformation is required.
- Demand planning and supply planning require sufficient historical data. Models need weeks to months of data before producing accurate forecasts.
- Inventory visibility aggregates data from multiple sources. Discrepancies often indicate data integration timing issues, not bugs.
- Connectors (SAP, EDI, etc.) have specific configuration requirements. Connector failures often stem from credential or network issues.
- S3 is the primary data exchange mechanism. Bucket permissions, file formats, and folder structures must match Supply Chain expectations.

## Anti-hallucination rules

1. Always cite specific instance IDs, flow names, or job IDs as evidence.
2. Data integration flows and data lake datasets are separate concepts. Never conflate them.
3. Demand planning requires historical data. Never claim instant forecast generation.
4. Inventory visibility aggregates data — discrepancies may be timing issues, not errors.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Instance | A1-A2 | Creation, configuration |
| B — Data | B1-B2 | Data integration failures, data lake issues |
| C — Planning | C1-C2 | Demand planning errors, supply planning issues |
| D — Visibility | D1-D2 | Inventory visibility, insights generation |
| E — Connectors | E1-E2 | Connector failures, S3 integration |
| F — Security | F1 | Permission issues |
| Z — Catch-All | Z1 | General troubleshooting |
