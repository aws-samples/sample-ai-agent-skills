---
name: iot-device-management-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS IoT Device Management
  problems by analyzing fleet provisioning, job execution, secure tunneling,
  thing groups, fleet indexing, device shadows, OTA updates, and following
  structured runbooks.
  Activate when: fleet provisioning failures, bulk registration errors,
  job execution issues, tunneling problems, thing group configuration,
  fleet indexing errors, device shadow sync failures, OTA update issues,
  or the user says something is wrong with IoT Device Management.
compatibility: >
  Requires AWS CLI or SDK access with iot, iot-jobs-data, iotsecuretunneling,
  iot-data, s3, iam, cloudwatch, and cloudtrail permissions.
---

# AWS IoT Device Management Diagnostics

## When to use

Any AWS IoT Device Management investigation — fleet provisioning, bulk registration, job execution, secure tunneling, thing groups, thing types, fleet indexing, device shadows, or OTA updates.

## Investigation workflow

### Step 1 — Collect and triage

```
aws iot list-things --query 'things[*].{Name:thingName,Type:thingTypeName,Version:version}'
aws iot list-thing-groups --query 'thingGroups[*].{Name:groupName,Arn:groupArn}'
aws iot list-jobs --status IN_PROGRESS --query 'jobs[*].{Id:jobId,Status:status,TargetSelection:targetSelection}'
```

### Step 2 — Domain deep dive

```
aws iot describe-thing --thing-name <name>
aws iot describe-job --job-id <job-id>
aws iotsecuretunneling list-tunnels --query 'tunnelSummaries[*].{Id:tunnelId,Status:status,Description:description}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iot.amazonaws.com --max-results 20
aws iot get-indexing-configuration
aws iot-data get-thing-shadow --thing-name <name> --output text | python -m json.tool
```

Read `references/guardrails.md` before concluding on any IoT Device Management issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `iot list-things` | List registered things |
| `iot describe-thing` | Get thing details |
| `iot list-jobs` | List IoT jobs |
| `iot describe-job` | Get job execution details |
| `iotsecuretunneling list-tunnels` | List secure tunnels |
| `iot get-indexing-configuration` | Check fleet indexing config |
| `iot-data get-thing-shadow` | Get device shadow state |
| `iot create-ota-update` | Create OTA update |

## Gotchas: AWS IoT Device Management

- Fleet provisioning templates use TWO flows: claim-based (devices use claim certificate) and trusted-user (admin provisions on behalf). They have different security models.
- Bulk registration requires a well-formed S3 JSON file. A single malformed line fails the entire batch unless error handling is configured.
- Job execution timeouts are separate from job-level timeouts. In-progress timers start when the device picks up the job, not when the job is created.
- Secure tunneling tokens expire after the configured lifetime (default 12 hours max). Tokens cannot be refreshed — a new tunnel must be opened.
- Thing groups support dynamic membership via fleet indexing queries. Dynamic groups require fleet indexing to be enabled FIRST.
- Device shadow sync conflicts occur when both device and cloud update the same shadow property simultaneously. The last writer wins — there is no automatic merge.
- OTA updates use code-signing. The signing profile, S3 bucket permissions, and IoT role must all be correctly configured or the update silently fails.

## Anti-hallucination rules

1. Always cite specific thing names, job IDs, tunnel IDs, or API responses as evidence.
2. Fleet provisioning claim certificates and production certificates serve different purposes. Never conflate them.
3. Job execution status on a device is independent of the job-level status. Never assume one from the other.
4. Secure tunneling uses WebSocket over TLS — it is NOT a VPN or SSH tunnel by itself. Never claim direct SSH without local proxy.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Fleet Provisioning | A1-A2 | Provisioning failures, bulk registration errors |
| B — Jobs | B1-B2 | Job execution failures, job timeout issues |
| C — Secure Tunneling | C1-C2 | Tunneling errors, tunnel connectivity |
| D — Thing Management | D1-D2 | Thing group issues, thing type configuration |
| E — Fleet Indexing & Shadows | E1-E2 | Fleet indexing errors, device shadow sync |
| F — OTA Updates | F1 | OTA update failures |
| Z — Catch-All | Z1 | General troubleshooting |
