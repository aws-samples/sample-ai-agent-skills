---
title: "F2 — Multi-Node Parallel Job Issues"
description: "Diagnose multi-node parallel job configuration and execution problems"
status: active
severity: MEDIUM
triggers:
  - "multi-node failed"
  - "MPI error"
  - "node communication"
  - "parallel job"
  - "main node"
owner: devops-agent
objective: "Identify and resolve multi-node parallel job issues"
context: "Multi-node parallel issues include: requires EC2 CE (not Fargate), node communication failures, main node coordination, placement group issues, or insufficient instances."
---

## Phase 1 — Triage

MUST:
- Check job definition for multi-node config: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].nodeProperties'`
- Verify CE is EC2 type (Fargate not supported)
- Check job status and node details: `aws batch describe-jobs --jobs <job-id>`
- Verify all nodes can communicate (security groups)

SHOULD:
- Check main node (index 0) logs
- Verify placement group configuration for low latency
- Check if sufficient instances are available for all nodes

MAY:
- Check ECS task placement for node distribution
- Review MPI/communication framework logs

## Phase 2 — Remediate

MUST:
- Use EC2 compute environment (not Fargate)
- Ensure security groups allow inter-node communication
- Configure node properties correctly in job definition

SHOULD:
- Use placement groups for low-latency communication
- Register multi-node job definition: `aws batch register-job-definition --job-definition-name <name> --type multinode --node-properties '{"numNodes":4,"mainNode":0,"nodeRangeProperties":[{"targetNodes":"0:3","container":{"image":"<image>","vcpus":4,"memory":8192}}]}'`

MAY:
- Implement node health checking
- Use EFA for high-performance networking

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure security groups allow inter-node communication | YELLOW | Network change — verify no unintended access |
| Configure node properties in job definition | GREEN | New definition revision — non-destructive |
| Use placement groups for low latency | GREEN | Infrastructure optimization — non-destructive |
| Use EFA for high-performance networking | YELLOW | Requires specific instance types and configuration |
| Implement node health checking | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Multi-node job failure affects HPC or distributed computing pipeline
- Node communication failures cause data loss in parallel processing
- Insufficient instances for all nodes block job execution

## Data Sensitivity

- **Classification: MEDIUM**
- Multi-node configuration reveals distributed computing architecture
- Node communication patterns expose data distribution strategy
- MPI/communication framework logs may contain processing details
- Placement group configuration reveals performance requirements

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest using Fargate for multi-node parallel jobs (not supported)
- **NEVER** suggest reducing node count below what the application requires
- **NEVER** suggest opening all ports between nodes without restricting to required protocols

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed security group rules for inter-node | Revert security group rules to previous state |
| Registered new multi-node job definition | Submit jobs using previous revision |
| Created placement group | Delete placement group if not needed |

## Output Format

```yaml
root_cause: "multi_node_issue — <specific_cause>"
evidence:
  - type: node_config
    content: "<multi-node configuration>"
  - type: node_status
    content: "<individual node status>"
severity: MEDIUM
mitigation:
  immediate: "Fix node configuration or communication"
  long_term: "Implement proper multi-node architecture with placement groups"
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
