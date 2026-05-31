---
title: "D2 — CPU Allocation"
description: "Diagnose enclave CPU allocation failures and performance issues"
status: active
severity: MEDIUM
triggers: ["CPU allocation failed", "not enough CPUs", "enclave slow", "CPU contention", "vCPU error"]
owner: devops-agent
objective: "Resolve CPU allocation issues for enclave creation and performance"
context: "Enclaves require dedicated vCPUs from the parent instance. Minimum is 2 vCPUs for the enclave. Parent must retain at least 2 vCPUs. CPU allocation is fixed at enclave start time."
---

## Phase 1 — Triage

MUST:
- Check instance vCPU count: `nproc`
- Check running enclave CPU allocation: `nitro-cli describe-enclaves --query '[*].{CPUCount:CpuCount}'`
- Verify instance type vCPU capacity: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].VCpuInfo'`

SHOULD:
- Check parent CPU utilization: `top -bn1 | head -5`
- Verify allocator CPU configuration: `cat /etc/nitro_enclaves/allocator.yaml`
- Check if enclave application is CPU-bound: review console output for performance indicators

MAY:
- Profile application CPU usage in Docker before enclave deployment
- Check CPU pinning configuration if applicable
- Review instance type options for more vCPUs

## Phase 2 — Remediate

MUST:
- Ensure --cpu-count in run-enclave is at least 2
- Ensure parent retains at least 2 vCPUs after enclave allocation
- Use an instance type with enough vCPUs for both parent and enclave

SHOULD:
- Allocate CPU count based on application workload requirements
- Configure allocator cpu_count in allocator.yaml to match enclave needs
- Restart allocator after configuration changes

MAY:
- Upgrade to a larger instance type for more vCPUs
- Optimize enclave application for better CPU efficiency
- Implement workload scheduling to manage CPU contention

## Safety Ratings

```
safety_ratings:
  - "nproc: GREEN — read-only CPU count"
  - "describe-enclaves: GREEN — read-only enclave info"
  - "ec2 describe-instance-types: GREEN — read-only instance info"
  - "run-enclave with cpu-count: YELLOW — dedicates CPUs to enclave"
  - "terminate-enclave: YELLOW — releases CPUs back to parent"
```

## Escalation Conditions

- Instance type has insufficient vCPUs for both parent and enclave workloads
- CPU-intensive enclave application requires more cores than available
- Parent instance services degraded due to CPU starvation
- Performance requirements cannot be met with current instance type

## Data Sensitivity

- `nproc`: LOW — returns CPU count
- `nitro-cli describe-enclaves`: LOW — returns CPU allocation
- `top`: LOW — returns process CPU usage
- `aws ec2 describe-instance-types`: LOW — returns instance specifications

## Prohibited Actions

- NEVER allocate all vCPUs to the enclave — parent needs at least 2
- NEVER set cpu-count below 2 for the enclave — it is the minimum requirement
- NEVER ignore CPU performance issues — they may indicate insufficient allocation
- NEVER change CPU allocation on a running enclave — it requires restart

## Phase 3 — Rollback

1. If enclave has too many CPUs allocated, terminate and restart with fewer
2. If parent is CPU-starved, terminate enclave to release CPUs
3. Restore allocator configuration to previous values
4. Verify parent instance services recover after CPU reallocation
5. Plan instance type upgrade if current type is insufficient

## Common Issues

- symptoms: "run-enclave fails with 'Not enough CPUs available'"
  diagnosis: "Instance does not have enough vCPUs for the requested enclave allocation plus parent minimum."
  resolution: "Reduce --cpu-count or use a larger instance type. Ensure total vCPUs >= enclave CPUs + 2 (parent minimum)."

- symptoms: "Enclave application runs slowly despite sufficient memory"
  diagnosis: "Enclave has too few vCPUs for the workload, or parent is competing for CPU resources."
  resolution: "Increase --cpu-count for the enclave. Reduce parent workload. Consider a larger instance type."

## Output Format

```yaml
root_cause: "cpu_allocation — <specific_cause>"
evidence:
  - type: cpu_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix CPU allocation"
  long_term: "Right-size instance type for combined parent and enclave workloads"
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
