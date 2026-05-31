---
title: "D2 — VPC Integration"
description: "Diagnose VPC integration issues with Internet Monitor"
status: active
severity: HIGH
triggers: ["VPC not monitored", "VPC integration failed", "VPC not showing data"]
owner: devops-agent
objective: "Resolve VPC integration issues with Internet Monitor"
context: "VPCs are a monitored resource type for Internet Monitor. VPC monitoring provides availability metrics for internet-bound traffic."
---

## Phase 1 — Triage

MUST:
- Check monitor resources: `aws internetmonitor get-monitor --monitor-name <name> --query 'Resources'`
- Verify VPC exists: `aws ec2 describe-vpcs --vpc-ids <vpc-id>`
- Confirm VPC ARN format: arn:aws:ec2:<region>:<account-id>:vpc/<vpc-id>

SHOULD:
- Verify the VPC has internet-bound traffic (NAT Gateway, Internet Gateway, or VPC endpoints)
- Check if the VPC is in the same region as the monitor
- Verify VPC flow logs for traffic confirmation

MAY:
- Check NAT Gateway metrics: `aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name ActiveConnectionCount --dimensions Name=NatGatewayId,Value=<id> --start-time <time> --end-time <time> --period 300 --statistics Sum`

## Phase 2 — Remediate

MUST:
- Add VPC to monitor if missing: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-add arn:aws:ec2:<region>:<account-id>:vpc/<vpc-id>`
- Ensure VPC has internet-bound traffic for Internet Monitor to analyze
- Verify monitor and VPC are in the same region

SHOULD:
- Allow time for data collection after adding VPC
- Verify ProcessingStatus transitions from COLLECTING_DATA to OK

MAY:
- Add multiple VPCs to a single monitor for consolidated monitoring
- Configure VPC flow logs for additional traffic visibility

## Common Issues

- symptoms: "VPC added to monitor but availability scores show INSUFFICIENT_DATA"
  diagnosis: "VPC may have no internet-bound traffic or only internal traffic."
  resolution: "Internet Monitor analyzes internet-bound traffic only. Verify the VPC has traffic flowing through an Internet Gateway or NAT Gateway. Internal VPC traffic is not analyzed."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "describe-vpcs: GREEN — read-only VPC details"
  - "update-monitor resources-to-add: YELLOW — adds resource to monitoring"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
```

## Escalation Conditions

- VPC consistently shows INSUFFICIENT_DATA despite confirmed internet-bound traffic
- VPC integration causes monitor ProcessingStatus errors
- Cross-region VPC monitoring not supported and alternative needed
- Multiple VPCs showing integration issues simultaneously

## Data Sensitivity

- `aws ec2 describe-vpcs`: LOW — returns VPC metadata
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metric data

## Prohibited Actions

- NEVER remove a VPC from a production monitor without explicit approval
- NEVER modify VPC networking configuration to troubleshoot Internet Monitor issues
- NEVER add VPCs from different regions to a monitor (not supported)
- NEVER assume VPC connectivity issues based solely on Internet Monitor data

## Phase 3 — Rollback

1. If a VPC was added in error, remove it: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-remove <arn>`
2. Revert any monitor configuration changes made during troubleshooting
3. Verify remaining monitored resources are still producing data
4. Document the VPC integration troubleshooting steps and outcome
5. Confirm no VPC networking configuration was inadvertently modified

## Output Format

```yaml
root_cause: "vpc_integration — <specific_cause>"
evidence:
  - type: integration_issue
    content: "<VPC and monitor details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC integration"
  long_term: "Automate VPC registration via IaC"
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
