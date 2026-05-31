---
name: groundstation-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Ground Station
  problems by analyzing satellite contacts, antenna configuration,
  data delivery, mission profiles, and ground station locations using
  structured runbooks. Activate when: contact scheduling failures,
  antenna configuration errors, data delivery issues, mission profile
  problems, or the user says something is wrong with Ground Station.
compatibility: >
  Requires AWS CLI or SDK access with GroundStation, IAM, EC2, S3,
  and CloudWatch permissions as applicable.
---

# AWS Ground Station Diagnostics

## When to use

Any AWS Ground Station investigation — satellite contacts, antenna configuration, data delivery, mission profiles, or ground station location troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws groundstation list-ground-stations
aws groundstation list-satellites
aws groundstation list-configs
aws groundstation list-mission-profiles
```

### Step 2 — Domain deep dive

```
aws groundstation get-config --config-id <id> --config-type <type>
aws groundstation get-mission-profile --mission-profile-id <id>
aws groundstation list-contacts --status SCHEDULED --start-time <start> --end-time <end>
aws groundstation describe-contact --contact-id <id>
```

### Step 3 — Detailed investigation

```
aws groundstation get-dataflow-endpoint-group --dataflow-endpoint-group-id <id>
aws cloudwatch get-metric-statistics --namespace AWS/GroundStation --metric-name ContactStatus --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=groundstation.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Ground Station issue.

## Anti-hallucination rules

1. Always cite specific contact IDs, config IDs, or API responses as evidence.
2. Contacts must be scheduled during available pass windows. Never assume on-demand.
3. Only onboarded satellites with mission profiles can be contacted.
4. Data is delivered to your AWS resources, not stored at ground stations.
5. Ground stations are at specific locations, not in all AWS regions.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Contacts | A1-A2 | Scheduling failures, contact execution |
| B — Antenna | B1-B2 | Configuration errors, tracking issues |
| C — Data Delivery | C1-C2 | Dataflow endpoints, S3 delivery |
| D — Mission Profiles | D1-D2 | Profile configuration, validation |
| E — Ground Stations | E1-E2 | Location selection, availability |
| Z — Catch-All | Z1-Z2 | General troubleshooting, satellite config |
