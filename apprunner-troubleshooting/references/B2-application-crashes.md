---
title: "B2 — App Runner Application Crashes"
description: "Diagnose application crashes, restarts, and unexpected terminations"
status: active
severity: HIGH
triggers:
  - "application crash"
  - "container restart"
  - "OOM"
  - "out of memory"
  - "segfault"
  - "exit code"
  - "application not responding"
owner: devops-agent
objective: "Identify why the App Runner application is crashing and stabilize the service"
context: "Application crashes in App Runner result in container restarts. Repeated crashes can cause the service to become unhealthy. Common causes include out-of-memory errors, unhandled exceptions, missing environment variables, dependency failures, and incompatible runtime configurations. App Runner does not support exec/SSH, so debugging relies on logs and metrics."
---

## Phase 1 — Triage

MUST:
- Check application logs for crash details: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- Check memory utilization: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name MemoryUtilization --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name CPUUtilization --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check instance configuration (CPU/memory): `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration'`
- Check active instances count: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name ActiveInstances --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 60 --statistics Average`

SHOULD:
- Check if crashes correlate with traffic spikes (concurrent requests)
- Verify all required environment variables and secrets are configured
- Check if the application has proper error handling and graceful shutdown

MAY:
- Check X-Ray traces for error patterns if tracing is enabled
- Review recent deployments that may have introduced the crash

## Phase 2 — Remediate

MUST:
- If OOM: increase instance memory (1 GB, 2 GB, 3 GB, or 4 GB options)
- If unhandled exception: fix the application code and redeploy
- If missing env var: add the required environment variable or secret to the service configuration

SHOULD:
- Implement proper error handling and logging in the application
- Add memory profiling to identify memory leaks
- Set appropriate instance size for the workload

MAY:
- Implement graceful shutdown handling (SIGTERM)
- Add circuit breakers for external dependency failures
- Enable X-Ray tracing for detailed error analysis

## Common Issues

- symptoms: "Application restarts repeatedly with no error in logs"
  diagnosis: "Out of memory — container killed by OOM killer before it can log the error."
  resolution: "Check MemoryUtilization metric. Increase instance memory or fix memory leaks."

- symptoms: "Application crashes with 'ECONNREFUSED' or connection errors"
  diagnosis: "Application tries to connect to a private resource without a VPC connector."
  resolution: "Attach a VPC connector if accessing private resources (see D2)."

- symptoms: "Application crashes on startup with 'missing environment variable'"
  diagnosis: "Required environment variable not configured in the service."
  resolution: "Add the missing environment variable via update-service or in the console."

## Output Format

```yaml
root_cause: "application_crash — <specific_cause>"
evidence:
  - type: application_logs
    content: "<crash log output>"
  - type: memory_utilization
    content: "<CloudWatch memory metric>"
  - type: instance_config
    content: "<CPU and memory allocation>"
severity: HIGH
mitigation:
  immediate: "Fix the crash cause and redeploy or increase instance resources"
  long_term: "Implement proper error handling, monitoring, and resource sizing"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase instance memory | YELLOW | Instance configuration change, requires redeployment |
| Fix unhandled exception in code | YELLOW | Application code change |
| Add missing environment variable | YELLOW | Service configuration change |
| Implement error handling | YELLOW | Application code change |
| Add memory profiling | GREEN | Diagnostic tooling |
| Enable X-Ray tracing | GREEN | Observability improvement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Application logs**: Crash details, stack traces — INTERNAL
- **Memory metrics**: Resource utilization data — INTERNAL
- **Instance configuration**: CPU and memory allocation — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER increase instance size without understanding the cost impact

## Phase 3 — Rollback

1. Revert instance configuration to previous CPU/memory settings
2. Revert application code changes and redeploy previous version
3. Remove any temporary environment variables added for debugging
4. Verify service stability with previous configuration
5. Monitor crash metrics for 30 minutes after rollback

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
