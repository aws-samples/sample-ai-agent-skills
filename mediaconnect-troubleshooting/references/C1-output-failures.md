---
title: "C1 — Output Failures"
description: "Diagnose MediaConnect output delivery failures"
status: active
severity: CRITICAL
triggers: ["output not delivering", "output failure", "no output signal", "output disconnected", "destination not receiving"]
owner: devops-agent
objective: "Restore output delivery to all configured destinations"
context: "Outputs deliver flow content to destinations. Each flow supports up to 50 outputs. Output failures can be protocol, network, or bandwidth related."
---

## Phase 1 — Triage

MUST:
- Check all outputs and their status: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Outputs[*].{Name:Name,OutputArn:OutputArn,Port:Port,Protocol:Transport.Protocol,Destination:Destination}'`
- Check output bitrate metric: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name OutputBitRate --dimensions Name=FlowARN,Value=<flow-arn> Name=OutputName,Value=<output-name> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`
- Verify flow is ACTIVE and source has bitrate

SHOULD:
- Check destination reachability from MediaConnect (firewall, security group)
- Verify protocol and port match between output and receiver
- Check OutputPacketLossPercent metric

MAY:
- Review output encryption settings if applicable
- Check if destination IP/port has changed

## Phase 2 — Remediate

MUST:
- Fix destination address if incorrect: `aws mediaconnect update-flow-output --flow-arn <flow-arn> --output-arn <output-arn> --destination <ip-address> --port <port>`
- Ensure receiver firewall allows inbound from MediaConnect IP range
- Verify protocol settings match between output and receiver

SHOULD:
- Check total output bandwidth does not exceed source bitrate
- Verify output encryption key matches receiver decryption key

MAY:
- Remove and recreate the output if update does not resolve: `aws mediaconnect remove-flow-output --flow-arn <flow-arn> --output-arn <output-arn>`
- Add a new output: `aws mediaconnect add-flow-outputs --flow-arn <flow-arn> --outputs Protocol=srt-caller,Port=<port>,Destination=<ip>`

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "update-flow-output: YELLOW — modifies output destination on a potentially live flow"
  - "remove-flow-output: RED — removes output, immediately stops delivery to that destination"
  - "add-flow-outputs: YELLOW — adds new output, consumes flow bandwidth"
```

## Escalation Conditions

- All outputs simultaneously fail while source is healthy
- Output shows connected but receiver reports no data
- Output packet loss exceeds 1% consistently
- Output failure correlates with specific time of day suggesting network congestion

## Data Sensitivity

- `aws mediaconnect describe-flow`: MEDIUM — contains destination IPs and encryption settings
- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values
- `aws mediaconnect update-flow-output`: MEDIUM — modifies destination IP and port
- `aws mediaconnect remove-flow-output`: HIGH — stops delivery to a destination

## Prohibited Actions

- NEVER remove a production output without confirming with downstream consumers
- NEVER change output destination IP without verifying the new destination is ready
- NEVER add outputs beyond the 50-output limit — check current count first
- NEVER disable output encryption on a production flow carrying protected content

## Phase 3 — Rollback

1. If output was removed, recreate it: `aws mediaconnect add-flow-outputs --flow-arn <flow-arn> --outputs Protocol=<protocol>,Port=<port>,Destination=<ip>`
2. If output destination was changed incorrectly, revert: `aws mediaconnect update-flow-output --flow-arn <flow-arn> --output-arn <output-arn> --destination <original-ip> --port <original-port>`
3. If a new output was added that causes bandwidth issues, remove it: `aws mediaconnect remove-flow-output --flow-arn <flow-arn> --output-arn <output-arn>`
4. Verify all remaining outputs are delivering via CloudWatch OutputBitRate metric
5. Document the output change and notify downstream consumers

## Common Issues

- symptoms: "Output bitrate is zero but flow source is healthy"
  diagnosis: "Destination is unreachable — firewall blocking, wrong IP, or receiver not listening."
  resolution: "Verify destination IP and port. Check receiver firewall allows inbound from MediaConnect. Confirm receiver is running."

- symptoms: "Output delivers but receiver shows corrupted video"
  diagnosis: "Protocol mismatch or encryption key mismatch between output and receiver."
  resolution: "Verify protocol settings match. For encrypted outputs, confirm encryption key/passphrase matches on both sides."

- symptoms: "Some outputs work but new output fails"
  diagnosis: "Total output bandwidth exceeds source bitrate, or output limit reached."
  resolution: "Check total output count (max 50). Verify aggregate output bandwidth fits within source bitrate."

## Output Format

```yaml
root_cause: "output_failure — <specific_cause>"
evidence:
  - type: output_metric
    content: "<OutputBitRate metric showing zero or drops>"
severity: CRITICAL
mitigation:
  immediate: "Restore output delivery"
  long_term: "Implement output monitoring with CloudWatch alarms"
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
