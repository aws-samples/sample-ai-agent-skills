---
title: "C1 — MPI Communication Errors"
description: "Diagnose MPI communication failures over EFA"
status: active
severity: HIGH
triggers: ["mpi error", "mpirun failed", "mpi communication timeout", "rank failed", "mpi abort"]
owner: devops-agent
objective: "Resolve MPI communication errors when using EFA as the transport layer"
context: "MPI (Open MPI, Intel MPI, MVAPICH2) uses libfabric to communicate over EFA. MPI configuration must specify the EFA provider."
---

## Phase 1 — Triage

MUST:
- Verify EFA connectivity between nodes: `fi_pingpong -p efa -e rdm <server-ip>` (run on both nodes)
- Check MPI configuration for EFA: verify `FI_PROVIDER=efa` is set
- Test basic MPI communication: `mpirun --np 2 --host <host1>,<host2> hostname`

SHOULD:
- Check MPI version and EFA compatibility: `mpirun --version`
- Verify all nodes have the same EFA driver and libfabric versions
- Check security group allows all self-traffic

MAY:
- Enable MPI debug output: `mpirun --mca btl_base_verbose 100 --np 2 <command>`
- Check for memory registration limits: `ulimit -l`

## Phase 2 — Remediate

MUST:
- Set EFA environment variables: `export FI_PROVIDER=efa` and `export FI_EFA_USE_DEVICE_RDMA=1` (for GDR)
- Configure MPI to use libfabric: `mpirun --mca mtl ofi --mca btl ^openib --np <N> <command>`
- Ensure all nodes can resolve each other's hostnames

SHOULD:
- Set memory lock limits: `ulimit -l unlimited` or configure in /etc/security/limits.conf
- Use the same MPI installation path on all nodes

MAY:
- Tune MPI parameters for EFA: `--mca mtl_ofi_provider_include efa`

## Safety Ratings

```
safety_ratings:
  - "fi_pingpong: GREEN — read-only connectivity test"
  - "mpirun hostname: GREEN — non-destructive MPI test"
  - "ulimit -l unlimited: YELLOW — changes resource limits for the session"
  - "mpirun with production workload: YELLOW — consumes compute resources"
```

## Escalation Conditions

- fi_pingpong succeeds but MPI communication fails
- MPI jobs hang indefinitely without error messages
- Communication works between some node pairs but not others
- MPI collective operations (allreduce) fail while point-to-point works

## Data Sensitivity

- `fi_pingpong` output: LOW — latency and bandwidth numbers
- MPI debug output: MEDIUM — may contain hostnames and network addresses
- `mpirun` output: MEDIUM — may contain application-specific data
- Host file contents: LOW — list of hostnames/IPs

## Prohibited Actions

- NEVER run MPI jobs as root unless absolutely required
- NEVER set ulimit -l unlimited system-wide without understanding memory implications
- NEVER disable EFA and fall back to TCP as a permanent solution
- NEVER ignore version mismatches between nodes — they cause subtle communication errors

## Phase 3 — Rollback

1. Revert MPI environment variables to defaults
2. If MPI configuration files were modified, restore from backup
3. If ulimit was changed, revert: `ulimit -l <original-value>`
4. Kill any hung MPI processes: `mpirun --host <hosts> killall -9 <process>`
5. Verify EFA connectivity is still functional: `fi_pingpong -p efa`

## Common Issues

- symptoms: "MPI job hangs during initialization with no error output"
  diagnosis: "Security group does not allow all self-traffic, blocking EFA OS-bypass communication."
  resolution: "Add inbound rule allowing all traffic from the same security group."

- symptoms: "MPI error: unable to find provider efa"
  diagnosis: "FI_PROVIDER is not set or libfabric EFA provider is not installed."
  resolution: "Set FI_PROVIDER=efa. Verify fi_info -p efa shows the EFA provider."

## Output Format

```yaml
root_cause: "mpi — <specific_cause>"
evidence:
  - type: mpi_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix MPI configuration or EFA connectivity"
  long_term: "Standardize MPI environment in container images or AMIs"
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
