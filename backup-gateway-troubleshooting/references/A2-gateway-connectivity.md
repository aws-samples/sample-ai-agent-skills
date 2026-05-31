---
title: "A2 — Gateway Connectivity"
description: "Diagnose gateway connectivity issues between on-premises and AWS"
status: active
severity: HIGH
triggers: ["gateway offline", "gateway not connected", "last seen time old", "connectivity lost"]
owner: devops-agent
objective: "Resolve gateway connectivity issues between on-premises environment and AWS"
context: "Gateway requires dual connectivity: to vCenter/ESXi (on-premises) and to AWS endpoints (cloud). Both paths must be functional."
---

## Phase 1 — Triage

MUST:
- Check gateway status and last seen time: `aws backup-gateway list-gateways --query 'Gateways[*].{Name:GatewayDisplayName,Arn:GatewayArn,LastSeen:LastSeenTime}'`
- Get gateway details: `aws backup-gateway get-gateway --gateway-arn <gateway-arn>`
- Verify the gateway VM is powered on and responsive in vCenter

SHOULD:
- Check network connectivity from gateway to AWS endpoints
- Verify DNS resolution is working on the gateway
- Check NTP synchronization

MAY:
- Review CloudWatch metrics for gateway: `aws cloudwatch get-metric-statistics --namespace AWS/BackupGateway --metric-name GatewayBytesUploaded --dimensions Name=GatewayArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check for recent gateway software updates

## Phase 2 — Remediate

MUST:
- Restore network connectivity to AWS endpoints (port 443)
- Fix DNS resolution if broken
- Restart the gateway VM if it is unresponsive

SHOULD:
- Verify proxy configuration if using a proxy
- Check for network infrastructure changes (firewall rules, routing)

MAY:
- Set up CloudWatch alarms for gateway connectivity
- Configure redundant network paths

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-gateways: GREEN — read-only gateway listing"
  - "backup-gateway get-gateway: GREEN — read-only gateway details"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "gateway VM restart: YELLOW — interrupts active backup/restore operations"
```

## Escalation Conditions

- Gateway has been offline for more than 1 hour
- Network infrastructure changes require network team involvement
- Gateway VM is unresponsive and cannot be restarted from vCenter
- Multiple gateways lose connectivity simultaneously

## Data Sensitivity

- `aws backup-gateway list-gateways`: LOW — returns gateway metadata
- `aws backup-gateway get-gateway`: LOW — returns gateway configuration
- Network diagnostics: MEDIUM — reveals network topology and routing

## Prohibited Actions

- NEVER restart the gateway VM during active backup or restore operations without warning
- NEVER modify network configuration without understanding the impact on other services
- NEVER delete the gateway to fix connectivity issues
- NEVER assume gateway offline means data loss — backup data is in AWS vaults

## Phase 3 — Rollback

1. If network changes were made, revert them if connectivity is not restored
2. If the gateway VM was restarted, verify it reconnects to AWS
3. Check that hypervisor connections are re-established after gateway restart
4. Verify pending backup jobs resume after connectivity is restored
5. Document the connectivity issue and resolution for future reference

## Common Issues

- symptoms: "Gateway LastSeenTime is hours old"
  diagnosis: "Gateway lost network connectivity to AWS endpoints."
  resolution: "Check gateway VM network, DNS, firewall rules. Verify port 443 access to AWS endpoints. Restart gateway VM if needed."

## Output Format

```yaml
root_cause: "gateway_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore network connectivity"
  long_term: "Set up connectivity monitoring and alerting"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
