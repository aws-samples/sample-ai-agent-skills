---
title: "Z1 — General VPN Advanced Troubleshooting"
description: "Catch-all runbook for advanced Site-to-Site VPN issues"
status: active
severity: MEDIUM
triggers: ["VPN error", "VPN not working", "site-to-site VPN issue", "VPN troubleshooting"]
owner: devops-agent
objective: "Systematic investigation of unclassified advanced VPN issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List all VPN connections: `aws ec2 describe-vpn-connections --query 'VpnConnections[*].{Id:VpnConnectionId,State:State,Type:Type,TgwId:TransitGatewayId,VgwId:VpnGatewayId}'`
- Check tunnel telemetry: `aws ec2 describe-vpn-connections --vpn-connection-ids <vpn-id> --query 'VpnConnections[*].VgwTelemetry'`
- Check customer gateways: `aws ec2 describe-customer-gateways --query 'CustomerGateways[*].{Id:CustomerGatewayId,State:State,BgpAsn:BgpAsn,IpAddress:IpAddress}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com --max-results 20`

SHOULD:
- Check VPN connection options (routing type, tunnel options, acceleration)
- Verify IAM permissions for the caller
- Check CloudWatch VPN metrics for anomalies

MAY:
- Check AWS Health Dashboard for VPN service issues
- Review VPN logs if enabled

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1–F1)

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for the identified issue category

## Common Issues

- symptoms: "General VPN connectivity issues"
  diagnosis: "Multiple possible causes: IKE mismatch, routing, DPD, or network path issues."
  resolution: "Follow systematic triage: check tunnel state → IKE logs → routing → DPD → network path."

## Safety Ratings

```
safety_ratings:
  - "describe-vpn-connections: GREEN — read-only VPN enumeration"
  - "describe-customer-gateways: GREEN — read-only CGW details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
```

## Escalation Conditions

- VPN service appears non-functional across multiple connections
- Multiple unrelated VPN issues occurring simultaneously
- CloudTrail shows unauthorized VPN API calls
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws ec2 describe-vpn-connections`: HIGH — reveals VPN configuration and tunnel details
- `aws ec2 describe-customer-gateways`: MEDIUM — reveals CGW IP addresses and BGP ASN
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make VPN configuration changes during general troubleshooting without first identifying the root cause
- NEVER delete VPN connections or customer gateways during investigation
- NEVER modify tunnel options as a troubleshooting step without understanding the impact
- NEVER dismiss the issue without documenting the investigation path and findings

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to the pre-investigation state
2. Verify all VPN connections are in expected states
3. Confirm tunnel telemetry shows expected UP/DOWN status
4. Document all investigation steps and findings for the network team
5. Verify VPN connectivity is functioning normally before closing the investigation

## Output Format

```yaml
root_cause: "general_vpn — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting"
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
