---
title: "F1 — Telemetry Issues"
description: "Diagnose Network Manager telemetry data gaps and collection failures"
status: active
severity: MEDIUM
triggers: ["telemetry missing", "no telemetry data", "bandwidth metrics missing", "latency data unavailable", "network manager dashboard empty"]
owner: devops-agent
objective: "Resolve Network Manager telemetry collection and display issues"
context: "Network Manager telemetry includes bandwidth, latency, and packet loss metrics. Data sources include VPN tunnel metrics, CloudWatch agents, and core network metrics."
---

## Phase 1 — Triage

MUST:
- Check Network Manager console for telemetry data availability
- Verify VPN tunnel status for VPN-based telemetry: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Check CloudWatch for Network Manager metrics: `aws cloudwatch list-metrics --namespace AWS/NetworkManager`

SHOULD:
- Verify the global network has registered resources
- Check if telemetry is available for specific attachment types
- Review IAM permissions for cloudwatch:GetMetricData

MAY:
- Check CloudWatch agent configuration on EC2 instances for custom telemetry
- Verify VPC Flow Logs are enabled for traffic analysis

## Phase 2 — Remediate

MUST:
- Ensure VPN tunnels are UP for VPN telemetry data
- Verify CloudWatch permissions in the IAM role
- Confirm resources are registered in the global network

SHOULD:
- Enable VPN tunnel metrics if not already enabled
- Configure CloudWatch agent for custom network metrics

MAY:
- Set up CloudWatch dashboards for Network Manager metrics
- Configure alarms for telemetry data gaps

## Common Issues

- symptoms: "No telemetry data in Network Manager dashboard"
  diagnosis: "No resources registered in the global network or VPN tunnels are DOWN."
  resolution: "Register resources in the global network and ensure VPN tunnels are UP."

- symptoms: "Telemetry data intermittent with gaps"
  diagnosis: "Telemetry propagation delay (up to 5 minutes) or VPN tunnel flapping."
  resolution: "Allow 5 minutes for data propagation. Check VPN tunnel stability if gaps persist."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN status"
  - "cloudwatch list-metrics: GREEN — read-only metric enumeration"
  - "cloudwatch get-metric-data: GREEN — read-only metric retrieval"
  - "cloudwatch put-metric-alarm: YELLOW — creates monitoring alarm"
```

## Escalation Conditions

- Telemetry data completely absent for all resources in the global network
- Telemetry shows anomalous values (e.g., 100% packet loss) not matching actual connectivity
- CloudWatch metrics for Network Manager not appearing in any region
- Telemetry gaps coincide with reported network performance issues

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration details
- `aws cloudwatch list-metrics`: LOW — returns metric names only
- `aws cloudwatch get-metric-data`: MEDIUM — reveals network performance characteristics

## Prohibited Actions

- NEVER modify VPN tunnel configuration to fix telemetry issues
- NEVER delete CloudWatch metrics or alarms during troubleshooting
- NEVER assume missing telemetry means a network outage — verify connectivity separately
- NEVER install CloudWatch agents on production instances without change management approval

## Phase 3 — Rollback

1. If CloudWatch alarms were created during troubleshooting, delete them if not needed: `aws cloudwatch delete-alarms --alarm-names <names>`
2. If CloudWatch agent configuration was modified, revert to previous configuration
3. Verify telemetry data collection is restored to pre-investigation state
4. Confirm no monitoring gaps were introduced during troubleshooting
5. Document telemetry configuration changes and notify the monitoring team

## Output Format

```yaml
root_cause: "telemetry — <specific_cause>"
evidence:
  - type: telemetry_state
    content: "<telemetry availability details>"
severity: MEDIUM
mitigation:
  immediate: "Fix telemetry data source"
  long_term: "Implement telemetry monitoring and gap alerting"
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
