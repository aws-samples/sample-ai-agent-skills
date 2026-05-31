---
title: "C1 — Snowball Edge Compute"
description: "Diagnose Snowball Edge compute and Lambda@Edge issues"
status: active
severity: HIGH
triggers: ["compute error", "Lambda on Snow", "edge compute failed", "compute instance error"]
owner: devops-agent
objective: "Resolve Snowball Edge compute issues including Lambda functions, compute resources, and service availability"
context: "Snowball Edge supports Lambda functions and EC2 instances for edge computing. Compute resources are limited by device type."
---

## Phase 1 — Triage

MUST:
- Check device compute resources: `snowballEdge describe-device --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- List running services: `snowballEdge list-services --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`
- Check compute service status: `snowballEdge describe-service --service-id ec2 --endpoint https://<device-ip>:8243 --manifest-file <manifest> --unlock-code <code>`

SHOULD:
- Verify device type supports compute (Storage Optimized with Compute or Compute Optimized)
- Check available compute capacity on the device

MAY:
- Review device logs for compute-related errors
- Check if Lambda functions were associated with the job during creation

## Phase 2 — Remediate

MUST:
- Start compute service if not running: `snowballEdge start-service --service-id ec2`
- Verify device type supports the required compute workload
- Ensure AMIs were included in the job creation for EC2 workloads

SHOULD:
- Monitor compute resource utilization
- Verify Lambda function packages are compatible with the Snow runtime

MAY:
- Optimize compute workloads for the available device resources

## Safety Ratings

```
safety_ratings:
  - "describe-device: GREEN — read-only device status"
  - "list-services: GREEN — read-only service listing"
  - "describe-service: GREEN — read-only service status"
  - "start-service: YELLOW — starts compute service on device"
```

## Escalation Conditions

- Compute service fails to start on a Compute Optimized device
- Device runs out of compute resources for required workloads
- Lambda functions fail with runtime errors specific to Snow environment
- Compute workload requires more resources than the device provides

## Data Sensitivity

- `snowballEdge describe-device`: MEDIUM — reveals device configuration and capacity
- `snowballEdge list-services`: LOW — reveals running services
- Compute workload logs: HIGH — may contain application data

## Prohibited Actions

- NEVER run compute workloads that exceed device capacity
- NEVER deploy untested Lambda functions to production Snow devices
- NEVER assume standard AWS Lambda runtime features are available on Snow
- NEVER ignore compute resource limits — overcommitting causes service failures

## Phase 3 — Rollback

1. Stop compute workloads that are causing issues
2. If EC2 instances were launched incorrectly, terminate them
3. Restart compute service if it became unresponsive
4. Verify data integrity after compute service restart
5. Document compute resource requirements for future deployments

## Common Issues

- symptoms: "EC2 service fails to start with 'InsufficientCapacity'"
  diagnosis: "Device does not have enough compute resources or is a Storage Optimized model without compute."
  resolution: "Verify device type. Use Compute Optimized or Storage Optimized with Compute for EC2 workloads."

## Output Format

```yaml
root_cause: "edge_compute — <specific_cause>"
evidence:
  - type: compute_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix compute issue"
  long_term: "Right-size device selection"
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
