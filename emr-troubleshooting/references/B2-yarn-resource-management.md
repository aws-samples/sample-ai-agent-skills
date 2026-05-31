---
title: "B2 — YARN Resource Management"
description: "Diagnose YARN resource allocation and contention issues"
status: active
severity: MEDIUM
triggers:
  - "YARN resource issue"
  - "container killed"
  - "pending containers"
  - "resource contention"
owner: devops-agent
objective: "Identify and resolve YARN resource management issues"
context: "YARN manages CPU and memory for all applications on EMR. Resource contention occurs when applications request more resources than available. Container kills happen when memory limits are exceeded. YARN scheduler configuration affects resource distribution."
---

## Phase 1 — Triage

MUST:
- Check YARN resource usage via ResourceManager UI or API
- Verify YARN configuration: yarn.nodemanager.resource.memory-mb, yarn.scheduler.maximum-allocation-mb
- Check for pending containers (resource starvation)
- Review container kill logs for memory violations

SHOULD:
- Check YARN scheduler type (capacity or fair)
- Verify queue configurations and resource limits
- Check for applications hogging resources
- Review node health status

MAY:
- Check YARN timeline server for historical data
- Review YARN audit logs

## Phase 2 — Remediate

MUST:
- Align application resource requests with YARN limits
- Fix memory settings to prevent container kills
- Ensure YARN nodemanager memory matches instance type

SHOULD:
- Configure YARN queues for workload isolation
- Set appropriate maximum allocation limits
- Monitor YARN metrics via CloudWatch

MAY:
- Implement YARN queue-based resource governance
- Create YARN configuration templates per instance type

## Common Issues

- symptoms: "Container killed by YARN for exceeding memory"
  diagnosis: "Application exceeds YARN container memory limit."
  resolution: "Increase yarn.scheduler.maximum-allocation-mb or reduce application memory request."

- symptoms: "Applications stuck in ACCEPTED state"
  diagnosis: "Insufficient YARN resources for the requested containers."
  resolution: "Scale cluster, reduce resource requests, or kill competing applications."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Align application resource requests with YARN limits | GREEN | Configuration tuning — non-destructive |
| Fix memory settings to prevent container kills | GREEN | Configuration tuning — applied on next run |
| Configure YARN queues for workload isolation | YELLOW | Changes resource distribution across applications |
| Kill competing applications | RED | Terminates running workloads — potential data loss |
| Scale cluster to add capacity | YELLOW | Adds nodes — cost increase |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- YARN resource contention affects multiple production applications
- Container kills cause data loss in long-running jobs

## Data Sensitivity

- **Classification: HIGH**
- YARN configuration reveals cluster resource allocation strategy
- Application resource usage exposes workload characteristics
- YARN audit logs may contain user and application details
- Queue configurations reveal multi-tenant resource governance

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest killing YARN applications without confirming with the user
- **NEVER** suggest reducing YARN nodemanager memory below safe operating thresholds

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed YARN configuration | Revert yarn-site.xml to previous values and restart NodeManagers |
| Configured YARN queues | Revert queue configuration to previous settings |
| Killed competing applications | Resubmit killed applications if they were needed |
| Scaled cluster (added nodes) | Remove added nodes if no longer needed |

## Output Format

```yaml
root_cause: "yarn_resources — <specific_cause>"
evidence:
  - type: yarn_config
    content: "<YARN configuration>"
  - type: resource_usage
    content: "<YARN resource utilization>"
severity: MEDIUM
mitigation:
  immediate: "Fix YARN resource allocation"
  long_term: "Implement YARN resource governance and monitoring"
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
