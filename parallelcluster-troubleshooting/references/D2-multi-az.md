---
title: "D2 — Multi-AZ Configuration Issues"
description: "Diagnose multi-AZ deployment problems in ParallelCluster"
status: active
severity: MEDIUM
triggers:
  - "multi-AZ"
  - "availability zone"
  - "cross-AZ"
  - "AZ placement"
owner: devops-agent
objective: "Resolve multi-AZ deployment and cross-AZ communication issues"
context: "ParallelCluster can deploy compute nodes across multiple AZs for capacity and resilience. Cross-AZ traffic incurs data transfer costs and adds latency. Shared storage must be accessible from all AZs."
---

## Phase 1 — Triage

MUST:
- Check queue subnet configuration in cluster config
- Verify subnets are in different AZs: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone}'`
- Check EFS mount targets exist in all AZs: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Verify compute nodes are launching in expected AZs: `aws ec2 describe-instances --filters "Name=tag:parallelcluster:cluster-name,Values=<cluster-name>" --query 'Reservations[*].Instances[*].{Id:InstanceId,AZ:Placement.AvailabilityZone}'`

SHOULD:
- Check for AZ-specific capacity issues
- Verify cross-AZ security group rules
- Review placement group configuration (if used)

MAY:
- Analyze cross-AZ data transfer costs
- Check for AZ-specific service quotas

## Phase 2 — Remediate

MUST:
- If mount targets missing: create EFS mount targets in all compute AZs
- If capacity in one AZ: configure multiple AZs in the queue
- If cross-AZ communication blocked: update security groups and NACLs

SHOULD:
- Use EFS for shared storage (multi-AZ native) instead of single-AZ FSx
- Configure placement groups for latency-sensitive workloads within a single AZ
- Monitor cross-AZ data transfer costs

MAY:
- Implement AZ-aware job scheduling
- Use separate queues per AZ for workload isolation

## Common Issues

- symptoms: "Compute nodes in AZ-b cannot mount shared storage"
  diagnosis: "EFS mount target missing in AZ-b."
  resolution: "Create EFS mount target in the AZ-b subnet."

- symptoms: "MPI jobs have high latency across nodes"
  diagnosis: "Nodes placed in different AZs — cross-AZ latency."
  resolution: "Use placement groups or single-AZ queue for latency-sensitive MPI jobs."

## Output Format

```yaml
root_cause: "multi_az — <specific_cause>"
evidence:
  - type: az_distribution
    content: "<node AZ placement>"
  - type: mount_targets
    content: "<EFS mount target AZs>"
severity: MEDIUM
mitigation:
  immediate: "Fix cross-AZ storage or networking"
  long_term: "Implement AZ-aware architecture for HPC workloads"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — multi-AZ configuration is operational
- Service Disruption Risk: MEDIUM — AZ issues affect compute capacity
- Compliance Impact: LOW — multi-AZ is a resilience concern

## Escalation Conditions
- Compute nodes fail to launch in specific AZs
- Cross-AZ latency affects job performance
- AZ capacity issues prevent scaling
- Multi-AZ configuration causes storage access issues
- AZ failure affects cluster availability

## Data Sensitivity
- AZ configurations — INTERNAL
- Subnet allocations — INTERNAL
- Capacity metrics per AZ — INTERNAL
- Cross-AZ traffic patterns — INTERNAL
- Never expose AZ-specific infrastructure details

## Prohibited Actions
- Do NOT configure multi-AZ without understanding cross-AZ latency impact
- Do NOT ignore AZ-specific capacity warnings
- Do NOT place all compute resources in a single AZ for production
- Do NOT modify AZ configurations during active jobs
- Do NOT bypass AZ health checks
- Do NOT ignore cross-AZ data transfer costs

## Phase 3 — Rollback
- Revert multi-AZ configuration changes
- Restore subnet allocations to previous state
- Re-enable AZ health monitoring
- Verify compute node distribution after changes
- Restore capacity reservations per AZ
- Document multi-AZ issues and configuration changes

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
