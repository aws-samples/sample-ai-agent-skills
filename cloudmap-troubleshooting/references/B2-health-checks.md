---
title: "B2 — Cloud Map Health Check Issues"
description: "Diagnose health check configuration and status problems"
status: active
severity: MEDIUM
triggers:
  - "health check failed"
  - "unhealthy instance"
  - "health check not working"
  - "custom health check"
owner: devops-agent
objective: "Identify and resolve health check issues affecting instance discovery"
context: "Cloud Map supports Route 53 health checks (public endpoints only) and custom health checks (application-managed). Health check failures cause instances to be excluded from discovery results."
---

## Phase 1 — Triage

MUST:
- Check service health check config: `aws servicediscovery get-service --id <service-id> --query 'Service.{HealthCheck:HealthCheckConfig,CustomHealth:HealthCheckCustomConfig}'`
- Check instance health status: `aws servicediscovery get-instances-health-status --service-id <service-id>`
- For Route 53 health checks: `aws route53 get-health-check-status --health-check-id <health-check-id>`
- Verify health check type matches endpoint accessibility (public vs private)

SHOULD:
- Check health check failure threshold and interval
- Verify health check path and port are correct
- Check if custom health check status is being updated by the application

MAY:
- Check Route 53 health check CloudWatch metrics
- Review health check history for patterns

## Phase 2 — Remediate

MUST:
- Use Route 53 health checks only for publicly accessible endpoints
- Use custom health checks for private endpoints
- Implement UpdateInstanceCustomHealthStatus calls in your application for custom health checks

SHOULD:
- Set appropriate failure threshold (default 1 for fast detection)
- Configure health check path to a lightweight health endpoint
- Monitor health check status changes

MAY:
- Set up CloudWatch alarms for health check failures
- Implement health check dashboards

## Common Issues

- symptoms: "All instances showing unhealthy"
  diagnosis: "Route 53 health check configured for private IP addresses."
  resolution: "Switch to custom health checks for private endpoints."

- symptoms: "Custom health check always shows initial status"
  diagnosis: "Application not calling UpdateInstanceCustomHealthStatus."
  resolution: "Implement health status reporting in your application."

- symptoms: "Health check flapping between healthy and unhealthy"
  diagnosis: "Health check threshold too low or endpoint intermittently unreachable."
  resolution: "Increase failure threshold or fix endpoint stability."

## Output Format

```yaml
root_cause: "health_check_issue — <specific_cause>"
evidence:
  - type: health_config
    content: "<health check type and configuration>"
  - type: health_status
    content: "<instance health statuses>"
  - type: health_check_details
    content: "<Route 53 health check results>"
severity: MEDIUM
mitigation:
  immediate: "Fix health check configuration or implement custom health reporting"
  long_term: "Implement comprehensive health monitoring across all services"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use Route 53 health checks for public endpoints | GREEN |
| Use custom health checks for private endpoints | GREEN |
| Implement UpdateInstanceCustomHealthStatus calls | GREEN |
| Set appropriate failure threshold | YELLOW |
| Configure health check path | GREEN |
| Set up CloudWatch alarms for health check failures | GREEN |

## Escalation Conditions

- Fix requires modifying production health check configurations
- Fix involves changing failure thresholds that affect instance availability

## Data Sensitivity

- **MEDIUM**: health check configurations expose endpoint paths and monitoring thresholds
- Health status data reveals service availability patterns

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If health check type was changed, restore previous configuration (may require service recreation)
2. If failure threshold was modified, restore previous threshold value
3. If custom health reporting was implemented, verify it correctly reports status
4. Verify rollback by confirming `aws servicediscovery get-instances-health-status` shows expected health states

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
