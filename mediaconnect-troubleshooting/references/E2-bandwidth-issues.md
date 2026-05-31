---
title: "E2 — Bandwidth Issues"
description: "Diagnose MediaConnect bandwidth constraints and bitrate problems"
status: active
severity: HIGH
triggers: ["bandwidth exceeded", "bitrate too high", "output quality degraded", "max bitrate", "throttling", "packet loss"]
owner: devops-agent
objective: "Resolve bandwidth constraints affecting flow quality and output delivery"
context: "MediaConnect flows have maximum bitrate settings. Total output bandwidth is constrained by source bitrate. Exceeding limits causes packet loss or quality degradation."
---

## Phase 1 — Triage

MUST:
- Check source bitrate: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average,Maximum`
- Check output bitrate per output: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name OutputBitRate --dimensions Name=FlowARN,Value=<flow-arn> Name=OutputName,Value=<output-name> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average,Maximum`
- Check flow configuration for max bitrate: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query '{MaxBitrate:Flow.Source.Transport.MaxBitrate,Outputs:Flow.Outputs[*].{Name:Name,Port:Port}}'`

SHOULD:
- Check SourcePacketLossPercent and OutputPacketLossPercent metrics
- Count total number of outputs and estimate aggregate bandwidth
- Verify source encoder bitrate matches flow expectations

MAY:
- Check MediaConnect service quotas for bandwidth limits
- Review historical bitrate trends for capacity planning

## Phase 2 — Remediate

MUST:
- If source exceeds max bitrate: reduce encoder output bitrate or increase flow max bitrate setting
- If total output bandwidth exceeds source: reduce number of outputs or lower individual output bitrates
- Address packet loss if bandwidth-related: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourcePacketLossPercent --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`

SHOULD:
- Set appropriate max bitrate on the flow source to match content requirements
- Use smoothing or CBR encoding at the source to avoid bitrate spikes
- Remove unused outputs to free bandwidth

MAY:
- Request service quota increase for higher bandwidth flows
- Consider splitting high-bandwidth delivery across multiple flows

## Safety Ratings

```
safety_ratings:
  - "get-metric-statistics: GREEN — read-only CloudWatch query"
  - "describe-flow: GREEN — read-only flow inspection"
  - "update-flow-source (max bitrate): YELLOW — changes bandwidth limit on potentially live flow"
  - "remove-flow-output: RED — removes output, stops delivery to that destination"
```

## Escalation Conditions

- Source bitrate consistently exceeds flow max bitrate causing packet loss
- All outputs experience quality degradation simultaneously
- Bandwidth issues correlate with specific times suggesting network congestion
- Service quota limits prevent increasing flow bandwidth to required levels

## Data Sensitivity

- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values
- `aws mediaconnect describe-flow`: MEDIUM — contains flow configuration details
- `aws mediaconnect update-flow-source`: MEDIUM — modifies source bandwidth settings
- `aws service-quotas get-service-quota`: LOW — returns quota values only

## Prohibited Actions

- NEVER reduce max bitrate below the source encoder output rate without coordinating with the encoder team
- NEVER remove outputs to free bandwidth without notifying downstream consumers
- NEVER ignore sustained packet loss — it indicates content quality degradation
- NEVER set max bitrate to an arbitrarily high value without understanding cost implications

## Phase 3 — Rollback

1. If max bitrate was changed, revert: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn>` with original max bitrate
2. If outputs were removed, recreate them: `aws mediaconnect add-flow-outputs --flow-arn <flow-arn> --outputs Protocol=<protocol>,Port=<port>,Destination=<ip>`
3. If encoder bitrate was changed, coordinate with encoder team to revert
4. Verify bitrate metrics return to expected levels after rollback
5. Document the bandwidth issue and update capacity planning

## Common Issues

- symptoms: "Source bitrate spikes cause intermittent packet loss"
  diagnosis: "Source encoder is using VBR encoding with peaks exceeding flow max bitrate."
  resolution: "Switch encoder to CBR or capped VBR. Increase flow max bitrate to accommodate peaks."

- symptoms: "Adding a new output causes existing outputs to degrade"
  diagnosis: "Total output bandwidth exceeds source bitrate capacity."
  resolution: "Remove the new output or reduce bitrate on existing outputs. Consider a separate flow for additional outputs."

- symptoms: "Consistent low bitrate despite high-quality source"
  diagnosis: "Max bitrate setting on the flow is too low, capping the source ingest."
  resolution: "Increase the max bitrate setting on the flow source to match the encoder output rate."

## Output Format

```yaml
root_cause: "bandwidth — <specific_cause>"
evidence:
  - type: bitrate_metric
    content: "<SourceBitRate and OutputBitRate metric data>"
severity: HIGH
mitigation:
  immediate: "Adjust bandwidth settings or reduce output count"
  long_term: "Implement capacity planning and bitrate monitoring alarms"
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
