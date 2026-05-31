---
title: "A1 — Accelerated VPN Issues"
description: "Diagnose AWS Accelerated Site-to-Site VPN creation and performance issues"
status: active
severity: HIGH
triggers: ["accelerated VPN not working", "VPN acceleration failed", "Global Accelerator VPN error", "accelerated VPN slow"]
owner: devops-agent
objective: "Resolve accelerated VPN creation, performance, and connectivity issues"
context: "Accelerated VPN uses AWS Global Accelerator to route traffic through the AWS global network. Only supported with transit gateway-based VPN connections. Cannot be added to existing connections."
---

## Phase 1 — Triage

MUST:
- Check VPN connection details and acceleration status: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].{Id:VpnConnectionId,Category:Category,TgwId:TransitGatewayId,Options:Options}'`
- Verify the VPN is TGW-based (not VGW): check TransitGatewayId is present
- Check tunnel telemetry: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`

SHOULD:
- Verify the region supports accelerated VPN
- Check CloudWatch metrics for tunnel performance: `aws cloudwatch get-metric-statistics --namespace AWS/VPN --metric-name TunnelDataIn --dimensions Name=VpnId,Value=<vpn-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify customer gateway is configured with the accelerated VPN tunnel endpoints (different from standard endpoints)

MAY:
- Compare latency metrics between accelerated and non-accelerated VPN connections
- Check Global Accelerator health in the AWS console

## Phase 2 — Remediate

MUST:
- If VPN is VGW-based: create a new TGW-based VPN connection with EnableAcceleration=true
- If acceleration was not enabled at creation: create a new VPN connection (cannot be modified)
- Ensure customer gateway is configured with the correct accelerated tunnel endpoints

SHOULD:
- Verify both tunnels are UP after customer gateway reconfiguration
- Test throughput and latency to confirm acceleration benefit

MAY:
- Set up CloudWatch alarms for accelerated VPN tunnel metrics
- Document the migration plan from non-accelerated to accelerated VPN

## Common Issues

- symptoms: "Cannot enable acceleration on existing VPN connection"
  diagnosis: "Acceleration must be specified at creation time and cannot be added later."
  resolution: "Create a new VPN connection with --options EnableAcceleration=true. Migrate traffic during a maintenance window."

- symptoms: "Accelerated VPN tunnels DOWN after creation"
  diagnosis: "Customer gateway configured with standard tunnel endpoints instead of accelerated endpoints."
  resolution: "Download the new VPN configuration and update the customer gateway with the accelerated tunnel endpoint IPs."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "create-vpn-connection (accelerated): YELLOW — creates new VPN resource with Global Accelerator"
  - "delete-vpn-connection: RED — removes VPN connectivity"
```

## Escalation Conditions

- Accelerated VPN creation fails in a region that should support it
- Accelerated VPN shows no latency improvement over standard VPN
- Customer gateway vendor does not support the accelerated tunnel endpoint format
- Migration from standard to accelerated VPN requires extended outage window

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals tunnel endpoints, PSK references, and CGW details
- `aws cloudwatch get-metric-statistics`: LOW — reveals throughput and tunnel state metrics
- VPN configuration download: HIGH — contains pre-shared keys and tunnel configuration

## Prohibited Actions

- NEVER attempt to modify an existing VPN connection to add acceleration
- NEVER delete the existing VPN connection before the accelerated replacement is verified
- NEVER share accelerated VPN tunnel endpoint IPs publicly
- NEVER assume accelerated VPN improves throughput — it improves latency and consistency

## Phase 3 — Rollback

1. If the accelerated VPN connection was created but is not working, keep the original VPN connection active
2. Delete the non-functional accelerated VPN: `aws ec2 delete-vpn-connection --vpn-connection-id <new-vpn-id>`
3. Verify the original VPN connection tunnels are still UP
4. Revert customer gateway configuration to the original VPN tunnel endpoints
5. Document the rollback and plan a retry with corrected configuration

## Output Format

```yaml
root_cause: "accelerated_vpn — <specific_cause>"
evidence:
  - type: vpn_config
    content: "<VPN connection and acceleration details>"
severity: HIGH
mitigation:
  immediate: "Fix accelerated VPN configuration"
  long_term: "Implement accelerated VPN with proper migration plan"
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
