---
title: "B3 — Global Accelerator Traffic Dial Issues"
description: "Diagnose traffic dial and endpoint weight configuration problems"
status: active
severity: MEDIUM
triggers:
  - "traffic dial not working"
  - "traffic percentage wrong"
  - "traffic distribution issues"
  - "endpoint weight"
owner: devops-agent
objective: "Resolve traffic dial and endpoint weight configuration issues"
context: "Traffic dial controls the percentage of traffic to an endpoint group (regional). Endpoint weights control distribution among endpoints within a group. These are different mechanisms."
---

## Phase 1 — Triage

MUST:
- Check traffic dial setting for each endpoint group: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query '{TrafficDialPercentage:EndpointGroup.TrafficDialPercentage,Region:EndpointGroup.EndpointGroupRegion}'`
- Check endpoint weights: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query 'EndpointGroup.EndpointDescriptions[*].{EndpointId:EndpointId,Weight:Weight,HealthState:HealthState}'`
- List all endpoint groups to compare traffic dials: `aws globalaccelerator list-endpoint-groups --listener-arn <listener-arn>`
- Check flow counts per region in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name NewFlowCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify client affinity is not overriding expected traffic distribution
- Check if all endpoint groups have healthy endpoints
- Test from multiple geographic locations to verify routing

MAY:
- Enable flow logs to analyze actual traffic distribution
- Check if DNS-based routing is affecting traffic patterns

## Phase 2 — Remediate

MUST:
- Set traffic dial to desired percentage (0-100): `aws globalaccelerator update-endpoint-group --endpoint-group-arn <endpoint-group-arn> --traffic-dial-percentage <percentage>`
- Set endpoint weights appropriately (0-255)

SHOULD:
- Ensure at least one endpoint group has traffic dial > 0%
- Verify endpoint health before adjusting traffic dial

MAY:
- Use traffic dial for blue/green deployments (gradually shift traffic between regions)

## Common Issues

- symptoms: "All traffic going to one region despite traffic dial settings"
  diagnosis: "Other endpoint groups have unhealthy endpoints, so traffic fails over."
  resolution: "Fix endpoint health in all regions before adjusting traffic dial."

- symptoms: "Traffic dial set to 50% but region gets more than 50%"
  diagnosis: "Traffic dial reduces traffic FROM a region; excess goes to nearest healthy region."
  resolution: "Understand that traffic dial diverts traffic away, not attracts it."

- symptoms: "Endpoint weight changes not reflected"
  diagnosis: "Client affinity (SOURCE_IP) is keeping existing connections on the same endpoint."
  resolution: "New connections will respect weights. Existing connections maintain affinity."

## Output Format

```yaml
root_cause: "traffic_dial — <specific_cause>"
evidence:
  - type: traffic_dial
    content: "<traffic dial percentages per endpoint group>"
  - type: endpoint_weights
    content: "<endpoint weights and health>"
  - type: flow_metrics
    content: "<CloudWatch flow count metrics>"
severity: MEDIUM
mitigation:
  immediate: "Adjust traffic dial or endpoint weights"
  long_term: "Monitor traffic distribution and endpoint health across regions"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
