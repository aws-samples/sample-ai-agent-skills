---
name: ivs-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Interactive Video
  Service (IVS) problems by analyzing channel creation, streaming failures,
  playback issues, chat, recording, low-latency, and real-time stages using
  structured runbooks.
  Activate when: channel creation failures, stream key issues, streaming
  failures, playback errors, chat problems, recording failures, low-latency
  issues, stage/real-time problems, or the user says something is wrong
  with IVS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with IVS, IVS Chat, IVS RealTime,
  S3, CloudWatch, CloudTrail permissions.
---

# IVS Diagnostics

## When to use

Any Amazon IVS investigation — channel creation, streaming, playback, chat, recording, low-latency optimization, or real-time stages.

## Investigation workflow

### Step 1 — Collect and triage

```
aws ivs list-channels --max-results 20
aws ivs get-channel --arn <channel-arn>
aws ivs list-streams --filter-by health=STREAM_LIVE
aws ivs get-stream --channel-arn <channel-arn>
aws cloudwatch get-metric-statistics --namespace AWS/IVS --metric-name ConcurrentViews --dimensions Name=Channel,Value=<channel-arn> --start-time <start> --end-time <end> --period 300 --statistics Maximum
```

### Step 2 — Domain deep dive

```
aws ivs list-stream-keys --channel-arn <channel-arn>
aws ivs get-stream-session --channel-arn <channel-arn> --stream-id <stream-id>
aws ivs list-recording-configurations
aws ivs-chat list-rooms --max-results 20
aws ivs-realtime list-stages --max-results 20
```

### Step 3 — Detailed investigation

```
aws ivs list-stream-sessions --channel-arn <channel-arn> --max-results 10
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ivs.amazonaws.com --max-results 10
aws s3 ls s3://<recording-bucket>/<path>/
```

Read `references/guardrails.md` before concluding on any IVS issue.

## Gotchas: IVS

- IVS has two modes: Low-Latency Streaming (channels) and Real-Time (stages). Different APIs.
- Stream keys are sensitive — treat like passwords. Regenerate if compromised.
- IVS auto-records to S3 if recording configuration is attached to channel.
- Low-latency mode uses WebRTC for sub-second playback via IVS Player SDK.
- Standard latency uses HLS and works with any HLS player.
- Chat is a separate service (ivs-chat) with its own rooms and tokens.
- IVS is available in limited regions — check availability before creating resources.

## Anti-hallucination rules

1. IVS Low-Latency and Real-Time are different products with different APIs.
2. Stream keys must be kept secret — never log or expose them.
3. IVS Player SDK is required for low-latency playback — standard HLS players add latency.
4. Recording requires S3 bucket in the same region as the IVS channel.
5. Chat tokens are short-lived — generate new tokens for each session.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Channel | A1-A2 | Channel creation, stream keys |
| B — Streaming | B1-B2 | Streaming failures, ingest issues |
| C — Playback | C1-C2 | Playback errors, quality |
| D — Chat | D1-D2 | Chat rooms, messaging |
| E — Recording | E1-E2 | Auto-recording, S3 config |
| F — Low-Latency | F1 | Low-latency optimization |
| G — Real-Time | G1-G2 | Stages, participant issues |
| Z — Catch-All | Z1 | General troubleshooting |
