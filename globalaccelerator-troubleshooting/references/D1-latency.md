---
title: "D1 — Global Accelerator Latency Issues"
description: "Diagnose high latency through Global Accelerator"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow response"
  - "latency increase"
  - "performance degradation"
owner: devops-agent
objective: "Identify and resolve latency issues in Global Accelerator traffic routing"
context: "Global Accelerator routes traffic over the AWS global network to reduce latency. High latency can be caused by suboptimal routing, unhealthy nearby endpoints, or endpoint-side performance issues."
---

## Phase 1 — Triage

MUST:
- Check processed bytes and flow metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name ProcessedBytesIn --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify endpoint health across regions: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query 'EndpointGroup.EndpointDescriptions[*].{EndpointId:EndpointId,HealthState:HealthState}'`
- Check traffic dial settings (may be routing to distant regions): `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query '{Region:EndpointGroup.EndpointGroupRegion,TrafficDial:EndpointGroup.TrafficDialPercentage}'`
- Test latency directly to the endpoint vs through GA: `curl -w "time_total: %{time_total}\n" -o /dev/null -s https://<ga-dns-name>/` and `curl -w "time_total: %{time_total}\n" -o /dev/null -s https://<endpoint-direct>/`

SHOULD:
- Check if the nearest endpoint group has healthy endpoints
- Verify client affinity is not routing to a distant endpoint
- Check endpoint application response times independently

MAY:
- Enable flow logs to analyze traffic paths
- Test from multiple geographic locations
- Check AWS Health Dashboard for network issues

## Phase 2 — Remediate

MUST:
- Ensure endpoint groups exist in regions closest to your users
- Fix unhealthy endpoints in nearby regions to prevent failover to distant regions
- Verify traffic dial is not diverting traffic away from nearby regions

SHOULD:
- Add endpoint groups in additional regions for better geographic coverage
- Optimize endpoint application performance
- Enable Origin Shield or caching at the endpoint level if applicable

MAY:
- Consider using custom routing for deterministic low-latency routing
- Evaluate if CloudFront should be used in combination for cacheable content

## Common Issues

- symptoms: "Latency higher through GA than direct connection"
  diagnosis: "Nearest endpoint group is unhealthy, traffic routed to distant region."
  resolution: "Fix endpoint health in the nearest region."

- symptoms: "Latency spikes at specific times"
  diagnosis: "Endpoint is overloaded during peak traffic."
  resolution: "Scale endpoint capacity or add endpoints to the group."

- symptoms: "Consistent high latency from specific geographic locations"
  diagnosis: "No endpoint group in a nearby region for those users."
  resolution: "Create endpoint groups in regions closer to affected users."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_comparison
    content: "<GA vs direct latency measurements>"
  - type: endpoint_health
    content: "<endpoint health per region>"
  - type: traffic_dial
    content: "<traffic dial settings>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint health or adjust traffic routing"
  long_term: "Deploy endpoints in regions closer to users"
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
