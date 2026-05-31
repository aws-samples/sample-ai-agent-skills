# AWS IoT FleetWise Diagnostics Skill

Agent skill for investigating and troubleshooting AWS IoT FleetWise problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS IoT FleetWise — fleet creation, vehicle models, decoder manifests, signal catalogs, campaigns, data collection, vehicle connectivity, edge agent, S3/Timestream destinations, and IAM configuration.

### Activate When

- Fleet creation failures
- Vehicle model or manifest errors
- Decoder manifest configuration issues
- Signal catalog problems
- Campaign creation or data collection failures
- Vehicle connectivity issues
- Edge agent errors
- S3 or Timestream destination failures
- IAM permission issues
- General FleetWise errors

---

## Skill Structure

```
iot-fleetwise-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-fleet-creation-failures.md
    ├── A2-vehicle-model-errors.md
    ├── B1-decoder-manifest-issues.md
    ├── B2-signal-catalog-errors.md
    ├── C1-campaign-creation-failures.md
    ├── C2-data-collection-issues.md
    ├── D1-vehicle-connectivity.md
    ├── D2-edge-agent-errors.md
    ├── E1-s3-destination-errors.md
    ├── E2-timestream-destination-errors.md
    ├── F1-iam-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Fleet & Vehicle** | A1–A2 | Fleet creation, vehicle models |
| **B — Manifests & Signals** | B1–B2 | Decoder manifests, signal catalogs |
| **C — Campaigns** | C1–C2 | Campaign creation, data collection |
| **D — Connectivity** | D1–D2 | Vehicle connectivity, edge agent |
| **E — Destinations** | E1–E2 | S3 destination, Timestream destination |
| **F — IAM** | F1 | IAM configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `iotfleetwise:*`, `s3:Get*`, `timestream:Describe*`, `iam:Get*`, `cloudwatch:GetMetricData` permissions

---

## License

MIT-0
