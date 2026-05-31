---
name: iot-fleetwise-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT FleetWise
  problems by analyzing fleet creation, vehicle models, decoder manifests,
  signal catalogs, campaigns, data collection, vehicle connectivity,
  edge agent, destinations, and IAM configuration following structured runbooks.
  Activate when: fleet creation failures, vehicle model errors, decoder
  manifest issues, signal catalog problems, campaign failures, data
  collection issues, vehicle connectivity problems, edge agent errors,
  destination configuration issues, or the user says something is wrong
  with IoT FleetWise.
compatibility: >
  Requires AWS CLI or SDK access with iotfleetwise, s3, timestream,
  iam, cloudwatch, and cloudtrail permissions.
---

# AWS IoT FleetWise Diagnostics

## When to use

Any AWS IoT FleetWise investigation — fleet creation, vehicle models, decoder manifests, signal catalogs, campaigns, data collection, vehicle connectivity, edge agent configuration, S3/Timestream destinations, or IAM setup.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iotfleetwise list-fleets --query 'fleetSummaries[*].{Id:id,Arn:arn,CreationTime:creationTime}'
aws iotfleetwise list-vehicles --query 'vehicleSummaries[*].{Name:vehicleName,ModelManifestArn:modelManifestArn}'
aws iotfleetwise list-campaigns --query 'campaignSummaries[*].{Name:name,Status:status,TargetArn:targetArn}'
```

### Step 2 — Domain deep dive

```
aws iotfleetwise get-vehicle --vehicle-name <name>
aws iotfleetwise get-campaign --name <campaign-name>
aws iotfleetwise get-decoder-manifest --name <manifest-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotfleetwise.amazonaws.com --max-results 20
aws iotfleetwise get-signal-catalog --name <catalog-name>
aws iotfleetwise list-decoder-manifest-signals --name <manifest-name>
```

Read `references/guardrails.md` before concluding on any IoT FleetWise issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `iotfleetwise list-fleets` | List vehicle fleets |
| `iotfleetwise get-vehicle` | Get vehicle details |
| `iotfleetwise get-campaign` | Get campaign configuration |
| `iotfleetwise get-decoder-manifest` | Get decoder manifest details |
| `iotfleetwise get-signal-catalog` | Get signal catalog definition |
| `iotfleetwise list-vehicles-in-fleet` | List vehicles in a fleet |
| `iotfleetwise get-logging-options` | Check FleetWise logging |

## Gotchas: AWS IoT FleetWise

- Vehicle models (model manifests) define the vehicle's signal structure. They must be in ACTIVE status before vehicles can be created from them. Draft manifests cannot be used.
- Decoder manifests translate between vehicle signals and the cloud representation. Each vehicle model needs a decoder manifest. Decoder manifests must also be ACTIVE.
- Signal catalogs are account-level resources — only ONE signal catalog per account per region. All vehicle models reference signals from this single catalog.
- Campaigns collect data based on conditions or time intervals. A campaign must target a fleet, not individual vehicles. Vehicles must be in the fleet to receive the campaign.
- Edge Agent (FWE) runs on the vehicle and collects data. It requires proper configuration, valid credentials, and network connectivity to send data to the cloud.
- S3 and Timestream destinations have different data formats and latency characteristics. S3 is for batch analytics; Timestream is for near-real-time queries.
- FleetWise uses a vision-based data model: signals → signal catalog → model manifest → decoder manifest → vehicle → fleet → campaign. Each layer depends on the previous.

## Anti-hallucination rules

1. Always cite specific vehicle names, campaign names, manifest ARNs, or API responses as evidence.
2. Signal catalogs are account-level singletons. Never suggest creating multiple signal catalogs in the same account/region.
3. Model manifests and decoder manifests have lifecycle states (DRAFT → ACTIVE). Never assume a draft manifest is usable.
4. Campaigns target fleets, not individual vehicles. Never suggest creating a campaign targeting a single vehicle directly.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Fleet & Vehicle | A1-A2 | Fleet creation failures, vehicle model errors |
| B — Manifests & Signals | B1-B2 | Decoder manifest issues, signal catalog errors |
| C — Campaigns | C1-C2 | Campaign creation failures, data collection issues |
| D — Connectivity | D1-D2 | Vehicle connectivity, edge agent errors |
| E — Destinations | E1-E2 | S3 destination errors, Timestream destination errors |
| F — IAM | F1 | IAM configuration |
| Z — Catch-All | Z1 | General troubleshooting |
