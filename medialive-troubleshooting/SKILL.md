---
name: medialive-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental MediaLive
  problems by analyzing channel creation, input configuration, output groups,
  encoding profiles, pipeline alerts, failover, and schedule actions using
  structured runbooks.
  Activate when: channel creation failures, input errors, output group issues,
  encoding problems, pipeline alerts, failover failures, schedule action errors,
  HLS/DASH output issues, input switching, audio configuration, caption problems,
  or the user says something is wrong with MediaLive without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MediaLive, MediaPackage, S3, IAM,
  CloudWatch, CloudTrail permissions.
---

# MediaLive Diagnostics

## When to use

Any AWS Elemental MediaLive investigation — channel failures, input configuration, output group issues, encoding profiles, pipeline alerts, failover, or schedule actions.

## Investigation workflow

### Step 1 — Collect and triage

```
aws medialive list-channels --max-results 20
aws medialive describe-channel --channel-id <id>
aws medialive list-inputs --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name ActiveAlerts --dimensions Name=ChannelId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws medialive describe-input --input-id <id>
aws medialive describe-input-security-group --input-security-group-id <id>
aws medialive describe-schedule --channel-id <id>
aws medialive list-input-devices --max-results 10
aws iam get-role --role-name <medialive-role>
```

### Step 3 — Detailed investigation

```
aws medialive describe-channel --channel-id <id> | jq '.PipelineDetails'
aws cloudwatch get-metric-statistics --namespace AWS/MediaLive --metric-name NetworkIn --dimensions Name=ChannelId,Value=<id>,Name=Pipeline,Value=PIPELINE_0 --start-time <start> --end-time <end> --period 60 --statistics Average
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=medialive.amazonaws.com --max-results 10
```

Read `references/guardrails.md` before concluding on any MediaLive issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-channel` | Channel config and state |
| `list-channels` | All channels and status |
| `describe-input` | Input details and endpoints |
| `list-inputs` | All inputs |
| `describe-schedule` | Schedule actions |
| `describe-input-security-group` | Input whitelist |
| `list-input-devices` | Elemental Link devices |
| `batch-start/stop` | Start/stop channels |

## Gotchas: MediaLive

- Channels have two pipelines (PIPELINE_0, PIPELINE_1) for redundancy. Both must be healthy.
- Input security groups whitelist CIDR ranges for PUSH inputs. Missing CIDRs cause silent drops.
- Channel class STANDARD has two pipelines; SINGLE_PIPELINE has one. Failover only works with STANDARD.
- MediaLive requires an IAM role with trust policy for medialive.amazonaws.com.
- Schedule actions are immutable once executed. You can only delete future actions.
- Output groups define destinations: MediaPackage, HLS, DASH, CMAF, Archive, Frame Capture, UDP.
- Input switching requires multiple inputs attached to the channel and schedule actions.
- Encoding profiles define video/audio codec settings per output. Changes require channel stop.

## Anti-hallucination rules

1. Always check both pipelines — a channel can appear running with one pipeline in alert state.
2. Never assume PUSH inputs are receiving data — verify source is sending to the correct endpoint.
3. Channel class determines pipeline count — SINGLE_PIPELINE cannot failover.
4. Schedule actions execute at the specified time and cannot be modified after execution.
5. Output locking group settings affect all outputs — verify segment length and GOP alignment.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Channel | A1-A3 | Creation failures, start/stop, state issues |
| B — Input | B1-B3 | Input config, security groups, switching |
| C — Output | C1-C3 | Output groups, HLS, DASH/CMAF |
| D — Encoding | D1-D2 | Video encoding, audio/captions |
| E — Pipeline | E1-E2 | Pipeline alerts, health monitoring |
| F — Failover | F1-F2 | Automatic failover, input failover |
| G — Schedule | G1 | Schedule actions |
| Z — Catch-All | Z1 | General troubleshooting |
