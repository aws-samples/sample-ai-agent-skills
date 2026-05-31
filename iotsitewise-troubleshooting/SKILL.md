---
name: iotsitewise-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT SiteWise problems
  by analyzing asset models, measurements, transforms, metrics, portals,
  gateways, data ingestion, and property value issues using structured
  runbooks. Activate when: asset model creation failures, measurement
  ingestion errors, transform computation issues, metric aggregation
  problems, portal access failures, gateway connectivity issues, or
  the user says something is wrong with IoT SiteWise.
compatibility: >
  Requires AWS CLI or SDK access with IoTSiteWise, IAM, CloudWatch,
  and Greengrass permissions as applicable.
---

# AWS IoT SiteWise Diagnostics

## When to use

Any AWS IoT SiteWise investigation — asset models, measurements, transforms, metrics, portals, gateways, data ingestion, or property value troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iotsitewise list-asset-models --max-results 25
aws iotsitewise describe-asset-model --asset-model-id <id>
aws iotsitewise list-assets --asset-model-id <id>
aws iotsitewise describe-asset --asset-id <id>
```

### Step 2 — Domain deep dive

```
aws iotsitewise list-gateways
aws iotsitewise describe-gateway --gateway-id <id>
aws iotsitewise describe-gateway-capability-configuration --gateway-id <id> --capability-namespace "iotsitewise:opcuacollector:2"
aws iotsitewise get-asset-property-value --asset-id <id> --property-id <prop-id>
aws iotsitewise batch-get-asset-property-value-history --entries '[{"assetId":"<id>","propertyId":"<prop-id>","entryId":"1"}]'
```

### Step 3 — Detailed investigation

```
aws iotsitewise list-portals
aws iotsitewise describe-portal --portal-id <id>
aws cloudwatch get-metric-statistics --namespace AWS/IoTSiteWise --metric-name BatchPutAssetPropertyValue.Success --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotsitewise.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any IoT SiteWise issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-asset-model` | Check model definition and properties |
| `describe-asset` | Check asset status and hierarchy |
| `get-asset-property-value` | Get latest property value |
| `describe-gateway` | Check gateway status and config |
| `describe-portal` | Check portal configuration |
| CloudWatch Metrics | Check ingestion rates and errors |

## Gotchas: AWS IoT SiteWise

- Asset models define the structure; assets are instances. Models have measurements (raw data), transforms (computed from measurements), and metrics (aggregated over time windows).
- Transforms compute in near real-time from other properties. They use a formula language with mathematical and comparison functions. Circular references cause model validation failures.
- Metrics aggregate data over time windows (1m, 5m, 1h, 1d). They can reference measurements, transforms, or other metrics. Metric computation has eventual consistency.
- Gateways run on Greengrass V2 and collect data from OPC-UA, Modbus, or Ethernet/IP sources. Gateway capability configuration defines the data sources.
- Portals use IAM Identity Center (SSO) for authentication. Portal users need explicit access to projects and assets.
- Data ingestion via BatchPutAssetPropertyValue has rate limits per property and per account. Timestamps must be within the allowed time range.

## Anti-hallucination rules

1. Always cite specific asset model IDs, property IDs, or API responses as evidence.
2. Transforms and metrics have different computation models. Never confuse them.
3. Gateway data sources require specific capability namespace configuration.
4. Portal access requires IAM Identity Center. Never suggest IAM user access.
5. Property data types must match between model definition and ingested data.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Asset Models | A1-A2 | Model creation, hierarchy issues |
| B — Properties | B1-B3 | Measurements, transforms, metrics |
| C — Portals | C1-C2 | Portal access, dashboard issues |
| D — Gateways | D1-D2 | Gateway connectivity, data source config |
| E — Ingestion | E1-E2 | Data ingestion, batch operations |
| F — Integration | F1-F2 | Alarms, notifications |
| Z — Catch-All | Z1 | General troubleshooting |
