---
title: "C1 — VPN CloudWatch Monitoring"
description: "Diagnose VPN CloudWatch metric gaps and monitoring configuration issues"
status: active
severity: MEDIUM
triggers: ["VPN metrics missing", "CloudWatch VPN not working", "VPN monitoring gaps", "TunnelState metric absent"]
owner: devops-agent
objective: "Resolve VPN CloudWatch monitoring and metric collection issues"
context: "AWS VPN publishes metrics to CloudWatch including TunnelState, TunnelDataIn, TunnelDataOut. Metrics are per-tunnel and per-VPN connection."
---

## Phase 1 — Triage

MUST:
- Check available VPN metrics: `aws cloudwatch list-metrics --namespace AWS/VPN --dimensions Name=VpnId,Value=<vpn-id>`
- Get tunnel state metric: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify the VPN connection exists and is not in DELETED state

SHOULD:
- Check per-tunnel metrics using TunnelIpAddress dimension
- Verify CloudWatch alarms are configured correctly: `aws cloudwatch describe-alarms --alarm-name-prefix vpn`
- Check IAM permissions for cloudwatch:GetMetricData

MAY:
- Review CloudWatch dashboard configuration for VPN widgets
- Check if custom metrics are being published alongside standard VPN metrics

## Phase 2 — Remediate

MUST:
- Verify the VPN connection ID is correct in metric queries
- Ensure the correct namespace (AWS/VPN) is used
- Check that the time range includes periods when the VPN was active

SHOULD:
- Create CloudWatch alarms for TunnelState: `aws cloudwatch put-metric-alarm --alarm-name vpn-tunnel-down --namespace AWS/VPN --metric-name TunnelState --dimensions Name=VpnId,Value=<vpn-id> --comparison-operator LessThanThreshold --threshold 1 --period 300 --evaluation-periods 2 --statistic Average --alarm-actions <sns-arn>`
- Set up alarms for both tunnels individually

MAY:
- Create a CloudWatch dashboard for VPN overview
- Set up composite alarms for multi-VPN monitoring

## Common Issues

- symptoms: "No VPN metrics in CloudWatch"
  diagnosis: "VPN connection is in DELETED state or the VPN ID is incorrect."
  resolution: "Verify the VPN connection exists and is AVAILABLE. Use the correct VPN connection ID."

- symptoms: "TunnelState shows 0 but VPN appears to work"
  diagnosis: "Metric delay of up to 5 minutes. The tunnel may have recovered since the last data point."
  resolution: "Allow 5 minutes for metric propagation. Check the most recent data points."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "cloudwatch put-metric-alarm: YELLOW — creates monitoring alarm"
  - "cloudwatch describe-alarms: GREEN — read-only alarm listing"
```

## Escalation Conditions

- VPN metrics completely absent for an active VPN connection
- CloudWatch alarms not triggering despite tunnel state changes
- Metric data inconsistent with actual tunnel status
- Cross-region VPN monitoring requires centralized dashboard

## Data Sensitivity

- `aws cloudwatch list-metrics`: LOW — returns metric names and dimensions
- `aws cloudwatch get-metric-statistics`: LOW — reveals tunnel state and throughput
- `aws cloudwatch describe-alarms`: LOW — returns alarm configuration

## Prohibited Actions

- NEVER delete existing CloudWatch alarms without confirming they are not used by incident management
- NEVER create alarms with overly sensitive thresholds that cause alert fatigue
- NEVER assume missing metrics means the VPN is down — check the VPN connection state directly
- NEVER modify alarm actions that notify production on-call teams without approval

## Phase 3 — Rollback

1. If CloudWatch alarms were created in error, delete them: `aws cloudwatch delete-alarms --alarm-names <names>`
2. If alarm thresholds were modified, revert to previous values
3. Verify existing monitoring is unaffected
4. Confirm alarm notifications are reaching the correct SNS topics
5. Document monitoring changes and notify the operations team

## Output Format

```yaml
root_cause: "vpn_cloudwatch — <specific_cause>"
evidence:
  - type: metric_state
    content: "<CloudWatch metric details>"
severity: MEDIUM
mitigation:
  immediate: "Fix CloudWatch monitoring configuration"
  long_term: "Implement comprehensive VPN monitoring dashboard"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
