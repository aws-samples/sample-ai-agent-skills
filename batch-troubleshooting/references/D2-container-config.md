---
title: "D2 — Container Configuration Issues"
description: "Diagnose container runtime configuration problems"
status: active
severity: MEDIUM
triggers:
  - "container config error"
  - "entrypoint wrong"
  - "environment variable missing"
  - "volume mount failed"
  - "container runtime"
owner: devops-agent
objective: "Identify and resolve container configuration issues"
context: "Container config issues include: wrong command/entrypoint, missing environment variables, volume mount errors, log configuration, or resource limits causing OOM."
---

## Phase 1 — Triage

MUST:
- Check container properties: `aws batch describe-job-definitions --job-definitions <name:revision> --query 'jobDefinitions[*].containerProperties'`
- Check container logs: `aws logs get-log-events --log-group-name /aws/batch/job --log-stream-name <stream>`
- Verify command and entrypoint
- Check environment variables

SHOULD:
- Verify volume mounts and mount points
- Check resource limits (memory, vCPU)
- Verify log configuration
- Check privileged mode and user settings

MAY:
- Test container locally with same configuration
- Check for ulimit settings

## Phase 2 — Remediate

MUST:
- Fix command/entrypoint in job definition
- Set required environment variables
- Fix volume mount paths

SHOULD:
- Register updated definition with correct container properties
- Set appropriate resource limits to prevent OOM

MAY:
- Implement container health checks
- Use secrets manager for sensitive environment variables

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix command/entrypoint in job definition | GREEN | New revision — non-destructive |
| Set required environment variables | GREEN | Configuration fix — non-destructive |
| Fix volume mount paths | GREEN | Configuration fix — non-destructive |
| Set appropriate resource limits | GREEN | Configuration tuning — non-destructive |
| Use secrets manager for sensitive variables | GREEN | Security improvement — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Container configuration errors block all job executions
- Volume mount failures cause data access issues
- Resource limits cause OOM kills on production jobs

## Data Sensitivity

- **Classification: MEDIUM**
- Container properties reveal application configuration and deployment
- Environment variables may contain sensitive configuration values
- Volume mounts reveal data access patterns and storage architecture
- Container logs may contain application output with sensitive data

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest running containers in privileged mode to resolve permission errors
- **NEVER** suggest hardcoding secrets in environment variables
- **NEVER** suggest disabling log configuration to hide errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Registered new job definition with fixed config | Submit jobs using previous revision if fix is incorrect |
| Changed environment variables | Revert to previous environment variable configuration |
| Modified volume mounts | Revert volume mount configuration |

## Output Format

```yaml
root_cause: "container_config_issue — <specific_cause>"
evidence:
  - type: container_properties
    content: "<container configuration>"
  - type: container_logs
    content: "<error output>"
severity: MEDIUM
mitigation:
  immediate: "Fix container configuration in job definition"
  long_term: "Implement container testing and validation"
```

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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
