---
title: "C1 — App Runner Auto-Scaling Issues"
description: "Diagnose auto-scaling configuration and behavior problems"
status: active
severity: MEDIUM
triggers:
  - "auto scaling"
  - "not scaling"
  - "too many instances"
  - "too few instances"
  - "max concurrency"
  - "scaling slow"
owner: devops-agent
objective: "Identify and fix App Runner auto-scaling configuration issues"
context: "App Runner auto-scaling is based on concurrent requests per instance (MaxConcurrency). When concurrent requests exceed MaxConcurrency × current instances, new instances are added up to MaxSize. Scaling is NOT based on CPU or memory utilization. MinSize (minimum 1) instances are always warm. Understanding the concurrency-based model is critical for proper scaling."
---

## Phase 1 — Triage

MUST:
- Check auto-scaling configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.AutoScalingConfigurationSummary'`
- Get detailed auto-scaling config: `aws apprunner describe-auto-scaling-configuration --auto-scaling-configuration-arn <asc-arn>`
- Check active instances: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name ActiveInstances --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check request count: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name RequestCount --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check request latency for signs of overload: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name RequestLatency --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics p99`

SHOULD:
- Compare concurrent request patterns with MaxConcurrency setting
- Check if MaxSize is limiting scale-out
- Verify the auto-scaling configuration is the latest revision

MAY:
- List all auto-scaling configurations: `aws apprunner list-auto-scaling-configurations`
- Check if the service is using the default auto-scaling configuration

## Phase 2 — Remediate

MUST:
- Adjust MaxConcurrency based on application capacity — lower values trigger more aggressive scaling
- Set MaxSize high enough to handle peak traffic
- Remember MinSize cannot be 0 — minimum is 1

SHOULD:
- Create a custom auto-scaling configuration instead of using the default
- Load test to determine optimal MaxConcurrency for the application
- Monitor ActiveInstances and RequestLatency to validate scaling behavior

MAY:
- Create multiple auto-scaling configurations for different traffic patterns
- Use CloudWatch alarms on RequestLatency to detect scaling issues early

## Common Issues

- symptoms: "Application is slow under load but instance count stays at 1"
  diagnosis: "MaxConcurrency is set too high (e.g., 200) so scaling threshold is never reached."
  resolution: "Lower MaxConcurrency to match actual application capacity (e.g., 50 or 25)."

- symptoms: "Too many instances running during low traffic"
  diagnosis: "MinSize is set higher than needed, keeping unnecessary warm instances."
  resolution: "Set MinSize to 1 (minimum). Scale-in happens automatically when traffic decreases."

- symptoms: "Scaling is too slow — latency spikes before new instances are ready"
  diagnosis: "Application has slow startup time. New instances take time to become healthy."
  resolution: "Optimize application startup time. Lower MaxConcurrency to trigger scaling earlier."

## Output Format

```yaml
root_cause: "auto_scaling — <specific_cause>"
evidence:
  - type: scaling_config
    content: "<MinSize, MaxSize, MaxConcurrency>"
  - type: active_instances
    content: "<CloudWatch ActiveInstances metric>"
  - type: request_metrics
    content: "<request count and latency>"
severity: MEDIUM
mitigation:
  immediate: "Adjust auto-scaling configuration parameters"
  long_term: "Load test and tune MaxConcurrency for optimal scaling behavior"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Adjust MaxConcurrency | YELLOW | Changes scaling behavior |
| Set MaxSize for peak traffic | YELLOW | Changes maximum capacity |
| Remember MinSize cannot be 0 | GREEN | Informational |
| Create custom auto-scaling config | YELLOW | New scaling configuration |
| Load test for optimal MaxConcurrency | GREEN | Non-production testing |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Scaling configuration**: MinSize, MaxSize, MaxConcurrency — INTERNAL
- **Instance metrics**: ActiveInstances, request counts — INTERNAL
- **Request latency**: Performance metrics — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER set MaxConcurrency to 1 in production (causes excessive scaling)

## Phase 3 — Rollback

1. Revert auto-scaling configuration to previous settings
2. Update service to use the previous auto-scaling configuration ARN
3. Monitor ActiveInstances and RequestLatency after rollback
4. Verify scaling behavior matches expected patterns
5. Delete the new auto-scaling configuration if it's no longer needed

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
