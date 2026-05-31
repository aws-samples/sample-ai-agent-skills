---
title: "E1 — Transport Stream Errors"
description: "Diagnose MPEG-TS transport stream errors in MediaConnect flows"
status: active
severity: HIGH
triggers: ["transport stream error", "TS error", "continuity counter", "PID error", "sync byte loss", "MPEG-TS"]
owner: devops-agent
objective: "Identify and resolve transport stream errors affecting video quality"
context: "Transport stream errors typically originate from the source encoder or network path. MediaConnect reports TS errors via CloudWatch metrics. Common errors include continuity counter errors, PID mapping issues, and sync byte loss."
---

## Phase 1 — Triage

MUST:
- Check TransportStreamError metric: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name TransportStreamError --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Sum`
- Check source bitrate stability: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average,Minimum,Maximum`
- Verify flow source details: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Source'`

SHOULD:
- Check SourcePacketLossPercent metric for network-related issues
- Compare error rate against baseline — some TS errors are normal at low rates
- Verify source encoder health and configuration

MAY:
- Capture a sample of the transport stream for offline analysis
- Check if errors correlate with specific times or events

## Phase 2 — Remediate

MUST:
- If errors are from source encoder: coordinate with encoder team to fix encoding issues
- If errors correlate with packet loss: address network path quality
- If errors are from protocol mismatch: verify source protocol settings match flow configuration

SHOULD:
- Use FEC (Forward Error Correction) for RTP sources over lossy networks
- Consider SRT protocol for sources over public internet (built-in error correction)
- Increase SRT latency if using SRT and seeing correctable errors

MAY:
- Switch to a more resilient protocol (SRT or Zixi) if currently using plain RTP
- Configure source failover to mitigate intermittent source-side TS errors

## Safety Ratings

```
safety_ratings:
  - "get-metric-statistics: GREEN — read-only CloudWatch query"
  - "describe-flow: GREEN — read-only flow inspection"
  - "update-flow-source (protocol change): RED — changes ingest protocol, requires sender reconfiguration"
  - "update-flow-source (FEC settings): YELLOW — modifies error correction, may affect latency"
```

## Escalation Conditions

- Transport stream errors exceed 1% of total packets consistently
- Errors cause visible artifacts in output video
- Errors correlate with packet loss but network path cannot be improved
- Source encoder team reports no issues but TS errors persist

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values
- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs and protocol details
- Transport stream analysis data: MEDIUM — may contain content metadata (program info, PID tables)
- `aws mediaconnect update-flow-source`: MEDIUM — modifies source configuration

## Prohibited Actions

- NEVER assume transport stream errors are caused by MediaConnect without checking source first
- NEVER change source protocol on a live production flow without a maintenance window
- NEVER ignore persistent TS errors — they indicate content quality degradation
- NEVER disable FEC to reduce latency without understanding the packet loss rate

## Phase 3 — Rollback

1. If protocol was changed, revert to original: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --protocol <original-protocol>`
2. If FEC settings were modified, revert: update source with original FEC column and row dimensions
3. If SRT latency was changed, revert: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn>` with original SRT settings
4. Verify TS error rate returns to baseline after rollback
5. Document the transport stream issue and coordinate with the encoder team

## Common Issues

- symptoms: "High TransportStreamError count with stable SourceBitRate"
  diagnosis: "Source encoder is producing malformed transport stream packets."
  resolution: "Check encoder configuration — verify PID mappings, continuity counters, and TS packet formatting."

- symptoms: "TransportStreamError spikes correlate with SourcePacketLossPercent spikes"
  diagnosis: "Network packet loss is causing transport stream discontinuities."
  resolution: "Enable FEC for RTP, increase SRT latency, or improve network path quality."

- symptoms: "Intermittent TS errors at regular intervals"
  diagnosis: "Encoder is inserting PAT/PMT tables or switching between programs causing brief discontinuities."
  resolution: "This may be normal behavior. Verify error rate is within acceptable threshold for the content type."

## Output Format

```yaml
root_cause: "transport_stream — <specific_cause>"
evidence:
  - type: ts_error_metric
    content: "<TransportStreamError metric data>"
severity: HIGH
mitigation:
  immediate: "Address source or network issue causing TS errors"
  long_term: "Implement FEC or SRT for error-resilient transport"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
