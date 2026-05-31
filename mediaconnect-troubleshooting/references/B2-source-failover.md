---
title: "B2 — Source Failover"
description: "Diagnose MediaConnect source failover configuration and operation issues"
status: active
severity: HIGH
triggers: ["failover not working", "source failover failed", "redundant source", "no failover", "source redundancy"]
owner: devops-agent
objective: "Ensure source failover is correctly configured and operational"
context: "Source failover requires two sources in different AZs with identical content. Failover mode can be MERGE or FAILOVER."
---

## Phase 1 — Triage

MUST:
- Check flow source and failover config: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query '{Source:Flow.Source,Sources:Flow.Sources,SourceFailoverConfig:Flow.SourceFailoverConfig}'`
- Verify both sources are in different AZs: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Sources[*].{Name:Name,IngestPort:IngestPort}'`
- Check bitrate on both sources: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> Name=SourceName,Value=<source-name> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`

SHOULD:
- Verify failover mode is set (MERGE or FAILOVER)
- Confirm both sources receive identical content from the encoder
- Check recovery window setting

MAY:
- Review SourceRecovery metric for failover events
- Verify encoder is sending to both MediaConnect ingest endpoints

## Phase 2 — Remediate

MUST:
- Enable failover if not configured: `aws mediaconnect update-flow --flow-arn <flow-arn> --source-failover-config FailoverMode=FAILOVER,RecoveryWindow=200,State=ENABLED`
- Ensure both sources are active and receiving content
- Verify sources are in different AZs

SHOULD:
- Set appropriate recovery window (100-15000 ms) based on content type
- Use MERGE mode for seamless failover with RTP content
- Verify encoder sends identical streams to both sources

MAY:
- Test failover by temporarily stopping one source feed
- Configure CloudWatch alarms on SourceBitRate per source

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "update-flow (failover config): YELLOW — modifies failover behavior on potentially live flow"
  - "get-metric-statistics: GREEN — read-only CloudWatch query"
  - "add-flow-sources: YELLOW — adds a second source, requires flow restart"
```

## Escalation Conditions

- Failover does not trigger when primary source drops despite correct configuration
- Both sources simultaneously lose connectivity
- Failover triggers but output shows glitches or content corruption
- Recovery window setting does not match content latency requirements

## Data Sensitivity

- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs and failover configuration
- `aws cloudwatch get-metric-statistics`: LOW — returns numeric metric values
- `aws mediaconnect update-flow`: MEDIUM — modifies flow failover behavior
- `aws mediaconnect add-flow-sources`: MEDIUM — adds source with IP and protocol details

## Prohibited Actions

- NEVER disable failover on a production flow without a maintenance window
- NEVER test failover by killing the primary source on a live production stream without stakeholder approval
- NEVER configure both sources in the same AZ — this defeats the purpose of failover
- NEVER set recovery window below 100ms — this causes excessive switching

## Phase 3 — Rollback

1. If failover was misconfigured, revert to previous settings: `aws mediaconnect update-flow --flow-arn <flow-arn> --source-failover-config FailoverMode=<original-mode>,RecoveryWindow=<original-window>,State=ENABLED`
2. If a second source was added incorrectly, remove it: `aws mediaconnect remove-flow-source --flow-arn <flow-arn> --source-arn <source-arn>`
3. If failover was disabled, re-enable: `aws mediaconnect update-flow --flow-arn <flow-arn> --source-failover-config State=ENABLED`
4. Verify primary source is delivering content after rollback
5. Document the failover configuration change and notify the media operations team

## Common Issues

- symptoms: "Failover does not trigger when primary source drops"
  diagnosis: "Failover may not be enabled, or both sources are in the same AZ."
  resolution: "Verify SourceFailoverConfig.State is ENABLED and sources are in different AZs."

- symptoms: "Failover triggers but output shows glitches"
  diagnosis: "Recovery window is too short or sources are not delivering identical content."
  resolution: "Increase recovery window. Verify encoder sends identical RTP streams with matching sequence numbers."

- symptoms: "Cannot add a second source to existing flow"
  diagnosis: "Flow was created without failover support or source limit reached."
  resolution: "Update flow to enable failover config, then add the second source."

## Output Format

```yaml
root_cause: "source_failover — <specific_cause>"
evidence:
  - type: failover_config
    content: "<SourceFailoverConfig details>"
severity: HIGH
mitigation:
  immediate: "Fix failover configuration"
  long_term: "Implement dual-AZ sources with automated monitoring"
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
