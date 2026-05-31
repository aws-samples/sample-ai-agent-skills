---
title: "G2 — EMR Serverless Capacity Issues"
description: "Diagnose EMR Serverless capacity and resource issues"
status: active
severity: MEDIUM
triggers:
  - "serverless capacity"
  - "cold start"
  - "resource limit exceeded"
  - "serverless throttling"
owner: devops-agent
objective: "Identify and resolve EMR Serverless capacity issues"
context: "EMR Serverless auto-provisions resources within application limits. Pre-initialized capacity reduces cold start. Resource limits control maximum CPU, memory, and disk. Service quotas limit concurrent applications and vCPUs."
---

## Phase 1 — Triage

MUST:
- Check application resource limits: `aws emr-serverless get-application --application-id <id> --query 'application.{MaxCapacity:maximumCapacity,AutoStart:autoStartConfiguration,AutoStop:autoStopConfiguration}'`
- Check service quotas: `aws service-quotas get-service-quota --service-code emr-serverless --quota-code <quota-code>`
- Verify pre-initialized capacity configuration
- Check for concurrent job resource contention

SHOULD:
- Monitor resource utilization during job runs
- Check for cold start delays
- Review auto-start/auto-stop configuration
- Verify network configuration for VPC-enabled applications

MAY:
- Check for account-level resource limits
- Review resource usage patterns

## Phase 2 — Remediate

MUST:
- Increase application resource limits if insufficient
- Request service quota increases if at limits
- Configure pre-initialized capacity for latency-sensitive workloads

SHOULD:
- Optimize job resource requests
- Configure auto-stop to release idle resources
- Monitor resource utilization trends

MAY:
- Implement resource usage dashboards
- Create capacity planning automation

## Common Issues

- symptoms: "Job takes long to start (cold start)"
  diagnosis: "No pre-initialized capacity configured."
  resolution: "Configure pre-initialized workers for the application."

- symptoms: "Job fails with resource limit exceeded"
  diagnosis: "Application max capacity too low for the workload."
  resolution: "Increase maximumCapacity settings on the application."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Increase application resource limits | YELLOW | Changes capacity — cost increase |
| Request service quota increases | GREEN | Quota request — no immediate impact |
| Configure pre-initialized capacity | YELLOW | Reserves resources — ongoing cost |
| Configure auto-stop to release idle resources | GREEN | Cost optimization — non-destructive |
| Optimize job resource requests | GREEN | Configuration tuning — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Capacity issues cause job delays affecting SLA
- Service quota limits block critical workloads

## Data Sensitivity

- **Classification: HIGH**
- Resource utilization metrics reveal workload characteristics and data volumes
- Application capacity configuration exposes processing requirements
- Pre-initialized capacity settings reveal latency sensitivity of workloads
- Service quota usage reveals organizational scale of data processing

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest setting unlimited resource limits without cost controls
- **NEVER** suggest disabling auto-stop on applications without cost justification

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Increased application resource limits | Reduce limits to previous values |
| Configured pre-initialized capacity | Remove pre-initialized capacity configuration |
| Changed auto-stop configuration | Revert auto-stop settings to previous values |

## Output Format

```yaml
root_cause: "serverless_capacity — <specific_cause>"
evidence:
  - type: resource_limits
    content: "<application resource limits>"
  - type: utilization
    content: "<resource utilization>"
severity: MEDIUM
mitigation:
  immediate: "Adjust resource limits or pre-initialized capacity"
  long_term: "Implement capacity planning and monitoring"
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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
