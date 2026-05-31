---
title: "A2 — Fleet Configuration"
description: "Diagnose fleet creation and configuration issues"
status: active
severity: HIGH
triggers: ["fleet creation failed", "fleet not scaling", "fleet configuration error", "worker count wrong"]
owner: devops-agent
objective: "Resolve fleet configuration issues for compute provisioning"
context: "Fleets provide compute workers for job execution. Service-managed fleets (SMF) are auto-scaled by AWS. Customer-managed fleets (CMF) use your EC2 instances with the worker agent."
---

## Phase 1 — Triage

MUST:
- List fleets: `aws deadline list-fleets --farm-id <farm-id> --query 'fleets[*].{FleetId:fleetId,DisplayName:displayName,Status:status,MaxWorkerCount:maxWorkerCount}'`
- Get fleet details: `aws deadline get-fleet --farm-id <farm-id> --fleet-id <fleet-id>`
- Check fleet status and worker count

SHOULD:
- For SMF: verify instance type, AMI, and subnet configuration
- For CMF: verify worker agent installation and registration
- Check queue-fleet associations: `aws deadline list-queue-fleet-associations --farm-id <farm-id>`

MAY:
- Review fleet scaling configuration and limits
- Check CloudWatch metrics for fleet utilization

## Phase 2 — Remediate

MUST:
- Fix fleet configuration (instance types, subnets, IAM role)
- For SMF: ensure subnets have internet access for agent communication
- For CMF: verify worker agent is installed and configured correctly

SHOULD:
- Configure appropriate min/max worker counts for workload
- Set up queue-fleet associations for job routing

MAY:
- Configure fleet capabilities for job matching
- Set up auto-scaling policies based on queue depth

## Safety Ratings

```
safety_ratings:
  - "list-fleets: GREEN — read-only fleet enumeration"
  - "get-fleet: GREEN — read-only fleet details"
  - "create-fleet: YELLOW — creates compute fleet with cost implications"
  - "update-fleet: YELLOW — changes fleet configuration, may affect running workers"
  - "delete-fleet: RED — terminates all workers and removes fleet"
```

## Escalation Conditions

- Service-managed fleet fails to launch instances due to EC2 capacity
- Fleet IAM role requires permissions changes managed by security team
- Fleet subnet configuration requires VPC changes
- Fleet scaling cannot meet job demand requiring architecture review

## Data Sensitivity

- Fleet configuration: MEDIUM — reveals compute architecture and instance types
- IAM role ARN: MEDIUM — fleet execution role identifier
- Subnet IDs: MEDIUM — reveals network architecture
- Worker details: LOW — worker IDs and status

## Prohibited Actions

- NEVER delete a fleet with running sessions without draining workers first
- NEVER modify fleet IAM role to have overly broad permissions
- NEVER set max worker count to unlimited without budget controls
- NEVER change fleet instance types during active rendering without testing

## Phase 3 — Rollback

1. If fleet configuration was changed incorrectly, update it: `aws deadline update-fleet --farm-id <id> --fleet-id <id>`
2. If fleet was deleted, recreate with same configuration and re-associate with queues
3. If scaling was changed, revert min/max worker counts
4. Verify queue-fleet associations are intact after changes

## Common Issues

- symptoms: "Fleet status is CREATE_FAILED"
  diagnosis: "Invalid configuration — wrong subnet, IAM role, or instance type."
  resolution: "Check fleet creation error. Verify subnet has internet access, IAM role has required permissions, instance type is valid."

- symptoms: "Fleet has workers but jobs stay PENDING"
  diagnosis: "No queue-fleet association or fleet capabilities don't match job requirements."
  resolution: "Create queue-fleet association. Verify fleet capabilities match job requirements."

## Output Format

```yaml
root_cause: "fleet_configuration — <specific_cause>"
evidence:
  - type: fleet_detail
    content: "<fleet configuration and status>"
severity: HIGH
mitigation:
  immediate: "Fix fleet configuration"
  long_term: "Implement fleet configuration as IaC"
```

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
