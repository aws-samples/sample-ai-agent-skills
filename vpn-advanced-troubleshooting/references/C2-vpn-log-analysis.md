---
title: "C2 — VPN Log Analysis"
description: "Diagnose VPN connection log configuration and analyze IKE/DPD log events"
status: active
severity: MEDIUM
triggers: ["VPN logs missing", "VPN log analysis", "IKE negotiation logs", "VPN log configuration error"]
owner: devops-agent
objective: "Resolve VPN log configuration issues and analyze tunnel negotiation events"
context: "VPN connection logs capture IKE negotiation, DPD events, and tunnel state changes. Logs are delivered to CloudWatch Logs with up to 5-minute delay. Data plane events are NOT logged."
---

## Phase 1 — Triage

MUST:
- Check if VPN logging is enabled: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].Options.TunnelOptions[*].LogOptions'`
- Check CloudWatch Logs for VPN log group: `aws logs describe-log-groups --log-group-name-prefix /aws/vpn`
- Query recent VPN log events: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --start-time <epoch-ms> --limit 50`

SHOULD:
- Filter for IKE errors: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --filter-pattern "ERROR" --limit 20`
- Filter for DPD events: `aws logs filter-log-events --log-group-name /aws/vpn/<vpn-id> --filter-pattern "DPD" --limit 20`
- Check IAM permissions for logs:FilterLogEvents

MAY:
- Set up CloudWatch Logs Insights queries for pattern analysis
- Export logs to S3 for long-term analysis

## Phase 2 — Remediate

MUST:
- Enable VPN logging if not configured: modify tunnel options to enable logging
- Verify the CloudWatch Logs log group exists and has correct retention
- Fix IAM permissions if log access is denied

SHOULD:
- Set appropriate log retention period: `aws logs put-retention-policy --log-group-name /aws/vpn/<vpn-id> --retention-in-days 30`
- Create CloudWatch Logs metric filters for IKE errors

MAY:
- Set up CloudWatch Logs Insights saved queries for common VPN log patterns
- Configure log subscription filters for real-time alerting

## Common Issues

- symptoms: "No VPN logs in CloudWatch Logs"
  diagnosis: "VPN logging not enabled on the tunnel options."
  resolution: "Enable logging: aws ec2 modify-vpn-tunnel-options --vpn-connection-id <vpn-id> --vpn-tunnel-outside-ip-address <ip> --tunnel-options EnableTunnelLogs=true."

- symptoms: "VPN logs show repeated IKE Phase 1 failures"
  diagnosis: "IKE parameters mismatch between AWS and customer gateway (encryption, hash, DH group)."
  resolution: "Compare IKE Phase 1 parameters on both sides. Align encryption algorithm, integrity algorithm, and DH group."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections (log options): GREEN — read-only configuration"
  - "logs describe-log-groups: GREEN — read-only log group listing"
  - "logs filter-log-events: GREEN — read-only log query"
  - "modify-vpn-tunnel-options (logging): YELLOW — enables logging, causes brief tunnel restart"
  - "logs put-retention-policy: YELLOW — changes log retention"
```

## Escalation Conditions

- VPN logs show persistent IKE authentication failures that cannot be resolved
- Log volume is excessive indicating continuous tunnel renegotiation
- VPN logging cannot be enabled due to service-linked role issues
- Log analysis reveals security-relevant events (unauthorized connection attempts)

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration
- `aws logs filter-log-events`: MEDIUM — reveals IKE negotiation details and tunnel events
- CloudWatch Logs Insights queries: MEDIUM — may reveal patterns in tunnel behavior

## Prohibited Actions

- NEVER share VPN log contents that include IKE negotiation details in public channels
- NEVER enable VPN logging on both tunnels simultaneously in production (causes brief restart)
- NEVER delete VPN log groups without confirming retention requirements are met
- NEVER assume VPN logs capture data plane issues — they only capture control plane events

## Phase 3 — Rollback

1. If VPN logging was enabled and caused issues, disable it on the affected tunnel
2. If log retention was modified, revert to the previous retention period
3. If metric filters were created, delete them if not needed
4. Verify VPN tunnel stability after any tunnel option modifications
5. Document log configuration changes

## Output Format

```yaml
root_cause: "vpn_log_analysis — <specific_cause>"
evidence:
  - type: log_events
    content: "<relevant VPN log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix VPN log configuration or address logged errors"
  long_term: "Implement VPN log monitoring with automated alerting"
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
