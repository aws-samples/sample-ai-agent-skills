---
title: "B1 — Global Accelerator Endpoint Health Check Failures"
description: "Diagnose endpoint health check failures in Global Accelerator"
status: active
severity: HIGH
triggers:
  - "endpoint unhealthy"
  - "health check failing"
  - "endpoint not receiving traffic"
owner: devops-agent
objective: "Identify why endpoints are failing Global Accelerator health checks"
context: "Global Accelerator performs independent health checks on endpoints. These are separate from ELB health checks. Unhealthy endpoints are removed from routing until they recover."
---

## Phase 1 — Triage

MUST:
- Check endpoint group and endpoint health: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn>`
- Verify health check configuration: `aws globalaccelerator describe-endpoint-group --endpoint-group-arn <endpoint-group-arn> --query '{HealthCheckPort:HealthCheckPort,HealthCheckProtocol:HealthCheckProtocol,HealthCheckPath:HealthCheckPath,HealthCheckIntervalSeconds:HealthCheckIntervalSeconds,ThresholdCount:ThresholdCount}'`
- Check endpoint health status in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/GlobalAccelerator --metric-name UnhealthyEndpointCount --dimensions Name=Accelerator,Value=<accelerator-arn> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Test the health check endpoint directly: `curl -sI http://<endpoint-ip>:<health-check-port><health-check-path>`

SHOULD:
- Compare GA health check config with ELB health check config if endpoint is an ALB/NLB
- Check security groups on the endpoint to ensure GA health check traffic is allowed
- Verify the endpoint is reachable from the internet (GA health checks come from the internet)

MAY:
- Check if the endpoint was recently added (initial health check may take time)
- Review VPC NACLs for the endpoint subnet

## Phase 2 — Remediate

MUST:
- Ensure the health check port is open on the endpoint's security group
- Verify the health check path returns HTTP 200 for HTTP/HTTPS health checks
- Align GA health check configuration with the actual endpoint service

SHOULD:
- Set appropriate health check interval and threshold count
- Ensure the endpoint application is running and responding on the health check port

MAY:
- Adjust health check protocol (TCP vs HTTP vs HTTPS) based on endpoint capabilities

## Common Issues

- symptoms: "All endpoints show as unhealthy"
  diagnosis: "Health check port is blocked by security group or NACL."
  resolution: "Allow inbound traffic on the health check port from 0.0.0.0/0 (GA health checks come from various IPs)."

- symptoms: "Endpoint healthy in ALB but unhealthy in GA"
  diagnosis: "GA health check port/path differs from ALB health check configuration."
  resolution: "Align GA health check settings with the ALB target group health check."

- symptoms: "Intermittent health check failures"
  diagnosis: "Health check threshold is too low or endpoint is under load."
  resolution: "Increase ThresholdCount to tolerate transient failures."

## Output Format

```yaml
root_cause: "endpoint_health — <specific_cause>"
evidence:
  - type: endpoint_health_status
    content: "<endpoint health details>"
  - type: health_check_config
    content: "<health check port, protocol, path>"
  - type: security_group
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix health check configuration or security group rules"
  long_term: "Align GA and ELB health checks, monitor endpoint health metrics"
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
