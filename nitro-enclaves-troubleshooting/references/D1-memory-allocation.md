---
title: "D1 — Memory Allocation"
description: "Diagnose enclave memory allocation failures and out-of-memory issues"
status: active
severity: HIGH
triggers: ["enclave out of memory", "memory allocation failed", "insufficient memory", "hugepages error", "OOM killed"]
owner: devops-agent
objective: "Resolve memory allocation issues for enclave creation and operation"
context: "Enclaves use hugepages allocated from the parent instance memory. The allocator service pre-reserves hugepages. Enclave memory is specified at run-enclave time and cannot be changed while running."
---

## Phase 1 — Triage

MUST:
- Check allocator configuration: `cat /etc/nitro_enclaves/allocator.yaml`
- Check available hugepages: `cat /proc/meminfo | grep -i huge`
- Check parent instance total memory: `free -m`

SHOULD:
- Verify allocator service is running: `systemctl status nitro-enclaves-allocator.service`
- Check EIF size to estimate minimum memory: `ls -lh <path-to-eif>`
- Review enclave console for OOM messages: `nitro-cli console --enclave-id <id>`

MAY:
- Check instance type memory limits: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[*].MemoryInfo'`
- Review dmesg for memory-related errors: `dmesg | grep -i "out of memory\|oom\|hugepage"`
- Profile application memory usage in Docker: `docker stats <container>`

## Phase 2 — Remediate

MUST:
- Configure allocator with sufficient memory in `/etc/nitro_enclaves/allocator.yaml`
- Restart allocator after configuration change: `sudo systemctl restart nitro-enclaves-allocator.service`
- Ensure --memory parameter in run-enclave is sufficient for application (EIF size + runtime overhead)

SHOULD:
- Leave at least 512MB and 2 vCPUs for the parent instance
- Set allocator memory to match the maximum enclave memory needed
- Optimize application memory usage to fit within available allocation

MAY:
- Use a larger instance type if more memory is needed
- Implement memory-efficient data processing in the enclave application
- Monitor enclave memory usage patterns for capacity planning

## Safety Ratings

```
safety_ratings:
  - "cat allocator.yaml: GREEN — read-only configuration check"
  - "cat /proc/meminfo: GREEN — read-only memory info"
  - "systemctl restart allocator: YELLOW — temporarily disrupts hugepage allocation"
  - "run-enclave with memory: YELLOW — reserves memory from parent"
```

## Escalation Conditions

- Instance type does not have enough memory for both parent and enclave workloads
- Allocator cannot reserve hugepages due to memory fragmentation
- Application memory requirements exceed instance type limits
- Memory allocation changes affect other services on the parent instance

## Data Sensitivity

- `/etc/nitro_enclaves/allocator.yaml`: LOW — contains memory configuration
- `/proc/meminfo`: LOW — contains system memory statistics
- `nitro-cli console`: MEDIUM — may show application memory usage details
- `docker stats`: LOW — contains container resource usage

## Prohibited Actions

- NEVER allocate all instance memory to the enclave — parent needs memory to operate
- NEVER reduce allocator memory while an enclave is running
- NEVER ignore OOM errors — they indicate the application needs more memory or optimization
- NEVER set hugepages to 0 if enclave operations are expected

## Phase 3 — Rollback

1. If allocator was misconfigured, restore previous settings and restart service
2. If enclave was started with too much memory, terminate and restart with correct allocation
3. If parent instance is memory-starved, terminate enclave to free memory
4. Verify parent instance services are healthy after memory reallocation
5. Document the correct memory allocation for the enclave workload

## Common Issues

- symptoms: "run-enclave fails with 'Not enough memory'"
  diagnosis: "Allocator has not reserved enough hugepages, or requested memory exceeds available hugepages."
  resolution: "Increase memory_mib in /etc/nitro_enclaves/allocator.yaml. Restart allocator service. Retry run-enclave."

- symptoms: "Enclave starts but application crashes with OOM"
  diagnosis: "Allocated memory is sufficient for EIF but not for application runtime data processing."
  resolution: "Increase --memory parameter. Profile application memory usage. Optimize data processing to use less memory."

## Output Format

```yaml
root_cause: "memory_allocation — <specific_cause>"
evidence:
  - type: memory_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix memory allocation"
  long_term: "Right-size instance and implement memory monitoring"
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
