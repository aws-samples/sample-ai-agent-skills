---
title: "E2 — Instance Fleet Issues"
description: "Diagnose EMR instance fleet provisioning problems"
status: active
severity: MEDIUM
triggers:
  - "instance fleet error"
  - "fleet provisioning failed"
  - "Spot capacity"
  - "instance fleet timeout"
owner: devops-agent
objective: "Identify and resolve instance fleet provisioning issues"
context: "Instance fleets allow specifying multiple instance types. EMR selects based on availability and price. On-Demand and Spot can be mixed. Spot interruptions affect task nodes. Fleet provisioning can fail if no instance types are available."
---

## Phase 1 — Triage

MUST:
- Check instance fleet status: `aws emr list-instance-fleets --cluster-id <id>`
- Verify instance type availability in the AZ
- Check for Spot capacity issues
- Review fleet provisioning timeout settings

SHOULD:
- Check Spot price history for selected instance types
- Verify On-Demand vs Spot allocation strategy
- Check for Spot interruption events
- Review fleet instance type diversity

MAY:
- Check EC2 Spot placement scores
- Review fleet allocation strategy

## Phase 2 — Remediate

MUST:
- Add more instance types to the fleet for flexibility
- Use On-Demand for core nodes to protect HDFS
- Ensure at least one instance type is available in the AZ

SHOULD:
- Diversify instance types across families and sizes
- Set appropriate On-Demand/Spot ratios
- Configure Spot allocation strategy (capacity-optimized recommended)

MAY:
- Use multiple AZs for better Spot availability
- Implement Spot interruption handling

## Common Issues

- symptoms: "Fleet stuck in PROVISIONING state"
  diagnosis: "No instance types available in the AZ."
  resolution: "Add more instance types. Try different AZs."

- symptoms: "Spot instances terminated mid-job"
  diagnosis: "Spot interruption due to capacity reclaim."
  resolution: "Use On-Demand for core. Use capacity-optimized Spot strategy. Enable checkpointing."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add more instance types to fleet | GREEN | Configuration change — improves flexibility |
| Use On-Demand for core nodes | YELLOW | Changes allocation — cost increase |
| Diversify instance types across families | GREEN | Configuration improvement — non-destructive |
| Configure Spot allocation strategy | GREEN | Optimization — non-destructive |
| Enable checkpointing for Spot resilience | GREEN | Adds data protection — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Spot interruptions cause repeated data loss on HDFS core nodes
- Fleet provisioning failure blocks time-sensitive workloads

## Data Sensitivity

- **Classification: HIGH**
- Instance fleet configuration reveals cost optimization strategy
- Spot price history and allocation strategy expose budget constraints
- Fleet provisioning patterns reveal workload scheduling
- Instance type selection reveals compute requirements for data processing

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest using Spot instances for core nodes with HDFS data that is not backed up to S3
- **NEVER** suggest removing all On-Demand capacity from production clusters

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed instance fleet configuration | Revert fleet instance types to previous configuration |
| Switched core nodes to On-Demand | Revert to Spot if cost is acceptable and data is on S3 |
| Changed Spot allocation strategy | Revert to previous allocation strategy |

## Output Format

```yaml
root_cause: "instance_fleet — <specific_cause>"
evidence:
  - type: fleet_status
    content: "<instance fleet provisioning status>"
  - type: capacity
    content: "<instance availability>"
severity: MEDIUM
mitigation:
  immediate: "Fix fleet configuration"
  long_term: "Diversify instance types and implement Spot best practices"
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
