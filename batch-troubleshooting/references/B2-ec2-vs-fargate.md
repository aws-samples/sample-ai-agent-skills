---
title: "B2 — EC2 vs Fargate Issues"
description: "Diagnose compute type selection and capability issues"
status: active
severity: MEDIUM
triggers:
  - "Fargate limitation"
  - "EC2 vs Fargate"
  - "compute type"
  - "GPU not available"
  - "Fargate resource limit"
owner: devops-agent
objective: "Identify and resolve compute type issues"
context: "EC2 and Fargate have different capabilities. Fargate: max 16 vCPU, 120 GB memory, no GPU, no multi-node. EC2: GPUs, custom AMIs, all instance types, multi-node parallel."
---

## Phase 1 — Triage

MUST:
- Check CE type: `aws batch describe-compute-environments --compute-environments <ce-name> --query 'computeEnvironments[*].computeResources.type'`
- Check job resource requirements vs compute type limits
- Verify GPU requirements (EC2 only)
- Check if multi-node parallel is needed (EC2 only)

SHOULD:
- Compare Fargate vCPU/memory combinations with job requirements
- Check Fargate Spot availability if using Fargate Spot
- Verify instance types for EC2 CE match workload needs

MAY:
- Compare cost between EC2 and Fargate for the workload
- Check for Fargate platform version compatibility

## Phase 2 — Remediate

MUST:
- Use EC2 CE for: GPU workloads, multi-node parallel, custom AMIs, >16 vCPU or >120 GB memory
- Use Fargate CE for: serverless, simple container jobs within resource limits
- Create appropriate CE type for workload requirements

SHOULD:
- For Fargate: use valid vCPU/memory combinations (0.25-16 vCPU, 0.5-120 GB)
- For EC2: configure appropriate instance types and allocation strategy

MAY:
- Use Fargate Spot for cost savings on fault-tolerant workloads
- Implement separate queues for EC2 and Fargate workloads

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create appropriate CE type for workload | GREEN | New resource creation — non-destructive |
| Configure Fargate vCPU/memory combinations | GREEN | Job definition change — non-destructive |
| Configure EC2 instance types and allocation | GREEN | CE configuration — non-destructive |
| Use Fargate Spot for fault-tolerant workloads | YELLOW | Spot may be interrupted — ensure retry strategy |
| Implement separate queues for EC2 and Fargate | GREEN | Architecture improvement — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Compute type mismatch blocks job execution (e.g., GPU needed but Fargate used)
- Migration between compute types requires job definition changes
- Fargate resource limits insufficient for workload

## Data Sensitivity

- **Classification: MEDIUM**
- CE type and configuration reveal infrastructure strategy
- Resource requirements expose workload characteristics
- Instance type selection reveals compute needs and cost optimization
- Fargate vs EC2 choice reveals operational preferences

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest using Fargate for GPU workloads (not supported)
- **NEVER** suggest using Fargate for multi-node parallel jobs (not supported)
- **NEVER** suggest exceeding Fargate resource limits (16 vCPU, 120 GB memory)

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created new CE with different type | Delete new CE if it doesn't meet requirements |
| Changed job definition for new compute type | Submit jobs using previous job definition revision |
| Created separate queues | Delete new queues if separation is not needed |

## Output Format

```yaml
root_cause: "compute_type_issue — <specific_cause>"
evidence:
  - type: ce_type
    content: "<compute environment type>"
  - type: resource_requirements
    content: "<job resource needs>"
severity: MEDIUM
mitigation:
  immediate: "Use correct compute type for workload requirements"
  long_term: "Design compute strategy matching workload characteristics"
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
