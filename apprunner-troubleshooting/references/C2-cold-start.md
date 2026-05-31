---
title: "C2 — App Runner Cold Start Latency"
description: "Diagnose and mitigate cold start latency when new instances are provisioned"
status: active
severity: MEDIUM
triggers:
  - "cold start"
  - "slow first request"
  - "startup latency"
  - "warm up"
  - "provisioning delay"
owner: devops-agent
objective: "Identify cold start causes and reduce latency for newly provisioned instances"
context: "Cold starts occur when App Runner provisions new instances to handle increased traffic. The new instance must pull the image, start the container, and pass health checks before serving traffic. MinSize instances are always warm, but instances beyond MinSize experience cold starts. Application startup time is the primary contributor to cold start latency."
---

## Phase 1 — Triage

MUST:
- Check auto-scaling configuration for MinSize: `aws apprunner describe-auto-scaling-configuration --auto-scaling-configuration-arn <asc-arn>`
- Check request latency percentiles: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name RequestLatency --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics p99`
- Check active instances over time to correlate scaling events with latency: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name ActiveInstances --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Review application startup logs for initialization time
- Check instance configuration (CPU/memory affects startup speed): `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration'`

SHOULD:
- Measure application startup time from container start to first successful health check
- Check if the application performs heavy initialization (database connections, cache warming, model loading)
- Verify health check settings allow enough time for startup

MAY:
- Profile application startup to identify slow initialization steps
- Check container image size — larger images take longer to pull

## Phase 2 — Remediate

MUST:
- Increase MinSize to keep more warm instances if cold starts are unacceptable
- Optimize application startup time — defer non-critical initialization
- Use smaller container images to reduce pull time

SHOULD:
- Implement lazy initialization for heavy resources (database connection pools, caches)
- Lower MaxConcurrency to trigger scaling earlier, before existing instances are overloaded
- Use multi-stage Docker builds to minimize image size

MAY:
- Implement a readiness endpoint that returns 200 only when the application is fully initialized
- Pre-warm caches and connections asynchronously after the health check passes
- Consider using ECR in the same region to minimize image pull latency

## Common Issues

- symptoms: "p99 latency spikes to 10+ seconds periodically"
  diagnosis: "Cold start during scale-out. New instances take time to start and pass health checks."
  resolution: "Increase MinSize to handle baseline traffic without scaling. Optimize startup time."

- symptoms: "First request after deployment is very slow"
  diagnosis: "Application performs heavy initialization on first request (lazy loading)."
  resolution: "Move initialization to startup. Implement eager loading for critical resources."

- symptoms: "Large container image causing slow cold starts"
  diagnosis: "Image size is several GB, causing slow pull times during scale-out."
  resolution: "Use multi-stage builds. Use alpine/slim base images. Remove unnecessary files."

## Output Format

```yaml
root_cause: "cold_start — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<p99 latency and correlation with scaling events>"
  - type: scaling_config
    content: "<MinSize and MaxConcurrency>"
  - type: startup_time
    content: "<application initialization duration>"
severity: MEDIUM
mitigation:
  immediate: "Increase MinSize or optimize application startup"
  long_term: "Minimize image size, implement lazy initialization, tune scaling parameters"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase MinSize | YELLOW | Increases always-warm instances (cost impact) |
| Optimize application startup time | YELLOW | Application code change |
| Use smaller container images | YELLOW | Image rebuild required |
| Implement lazy initialization | YELLOW | Application code change |
| Lower MaxConcurrency | YELLOW | Changes scaling trigger point |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Latency metrics**: p99 latency and scaling correlation — INTERNAL
- **Scaling configuration**: MinSize and MaxConcurrency — INTERNAL
- **Startup time**: Application initialization duration — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER set MinSize higher than needed without cost approval

## Phase 3 — Rollback

1. Revert MinSize to previous value
2. Revert MaxConcurrency to previous value
3. Revert application code changes if startup optimization caused issues
4. Revert container image to previous version if image changes caused problems
5. Monitor p99 latency to confirm rollback resolved the issue

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
