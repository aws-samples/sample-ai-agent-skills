---
title: "E2 — Network Health"
description: "Diagnose overall Private 5G network health issues"
status: active
severity: HIGH
triggers: ["network unhealthy", "network degraded", "health check failed", "network alarm"]
owner: devops-agent
objective: "Resolve overall network health degradation and monitoring issues"
context: "Network health encompasses radio unit status, device connectivity, throughput, and VPC integration health."
---

## Phase 1 — Triage

MUST:
- Get network status: `aws private-networks get-network --network-arn <arn>`
- List all network resources and check statuses: `aws private-networks list-network-resources --network-arn <arn>`
- Check CloudWatch alarms: `aws cloudwatch describe-alarms --alarm-name-prefix Private5G`

SHOULD:
- Review network site statuses: `aws private-networks list-network-sites --network-arn <arn>`
- Check connected device count trends: `aws cloudwatch get-metric-statistics --namespace AWS/Private5G --metric-name ConnectedDevices --dimensions Name=NetworkArn,Value=<arn> --start-time <start> --end-time <end> --period 3600 --statistics Average`
- Review CloudTrail for recent changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=private-networks.amazonaws.com --max-results 20`

MAY:
- Check AWS Health Dashboard for Private 5G service issues
- Review VPC flow logs for traffic anomalies

## Phase 2 — Remediate

MUST:
- Address any radio units in non-ACTIVE status
- Resolve network site issues identified in triage
- Fix VPC connectivity if ENI health checks fail

SHOULD:
- Set up CloudWatch alarms for key health metrics
- Implement automated monitoring for radio unit status changes
- Create a network health dashboard

MAY:
- Enable VPC flow logs for ongoing traffic monitoring
- Configure SNS notifications for health alerts

## Safety Ratings

```
safety_ratings:
  - "get-network: GREEN — read-only network status"
  - "list-network-resources: GREEN — read-only resource enumeration"
  - "describe-alarms: GREEN — read-only alarm status"
  - "cloudtrail lookup-events: GREEN — read-only audit log"
```

## Escalation Conditions

- Network status is not AVAILABLE
- Multiple radio units offline simultaneously
- Network health degradation correlates with AWS service events
- Connected device count drops to zero unexpectedly

## Data Sensitivity

- `aws private-networks get-network`: LOW — returns network metadata
- `aws cloudwatch describe-alarms`: LOW — returns alarm configurations
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals
- VPC flow logs: MEDIUM — reveals network traffic patterns

## Prohibited Actions

- NEVER delete or recreate the network as a health fix
- NEVER disable CloudWatch alarms to suppress health warnings
- NEVER ignore radio unit status changes without investigation
- NEVER modify network configuration without a maintenance window

## Phase 3 — Rollback

1. If monitoring configuration was changed, restore original alarm settings
2. If network resources were modified, verify all return to healthy status
3. Confirm device connectivity is restored across all sites
4. Verify CloudWatch metrics show normal patterns
5. Document the health incident timeline and resolution

## Common Issues

- symptoms: "Network shows AVAILABLE but no devices can connect"
  diagnosis: "Radio units may be offline or SAS authorization expired."
  resolution: "Check individual radio unit statuses. Verify SAS registration."

- symptoms: "CloudWatch alarms firing for Private 5G metrics"
  diagnosis: "Network health metric crossed alarm threshold."
  resolution: "Investigate the specific metric in alarm. Follow the corresponding runbook for that issue domain."

## Output Format

```yaml
root_cause: "network_health — <specific_cause>"
evidence:
  - type: health_status
    content: "<health data>"
severity: HIGH
mitigation:
  immediate: "Restore network health"
  long_term: "Implement comprehensive health monitoring"
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
