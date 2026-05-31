---
title: "C2 — NCCL Errors"
description: "Diagnose NCCL collective communication errors over EFA"
status: active
severity: CRITICAL
triggers: ["nccl error", "nccl timeout", "allreduce failed", "nccl init failed", "distributed training failed"]
owner: devops-agent
objective: "Resolve NCCL communication errors for distributed GPU training over EFA"
context: "NCCL requires the aws-ofi-nccl plugin to use EFA. Without it, NCCL falls back to TCP sockets with dramatically lower performance."
---

## Phase 1 — Triage

MUST:
- Verify aws-ofi-nccl plugin is installed: `ls /opt/amazon/efa/lib64/libnccl-net-ofi.so` or check NCCL debug output
- Check NCCL environment: `echo $NCCL_DEBUG $NCCL_NET $FI_PROVIDER`
- Verify EFA is functional: `fi_info -p efa`

SHOULD:
- Enable NCCL debug logging: `export NCCL_DEBUG=INFO` and rerun the training job
- Check GPU driver and CUDA version compatibility: `nvidia-smi`
- Verify all nodes have identical NCCL, aws-ofi-nccl, and libfabric versions

MAY:
- Run NCCL test suite: `nccl-tests/build/all_reduce_perf -b 8 -e 256M -f 2 -g <gpus-per-node>`
- Check for GPUDirect RDMA support: `lsmod | grep nvidia_peermem`

## Phase 2 — Remediate

MUST:
- Install aws-ofi-nccl plugin if missing (included in EFA installer with -g flag)
- Set required environment variables: `export NCCL_NET=ofi` and `export FI_PROVIDER=efa`
- For multi-EFA instances: `export FI_EFA_SET_CUDA_SYNC_MEMOPS=0`

SHOULD:
- Enable GPUDirect RDMA: `export FI_EFA_USE_DEVICE_RDMA=1` and `export NCCL_NET_GDR_LEVEL=PHB`
- Set NCCL socket interface: `export NCCL_SOCKET_IFNAME=eth0`
- Configure NCCL topology: `export NCCL_TOPO_FILE=/opt/amazon/efa/share/aws-ofi-nccl/xml/p4d-24xl-topo.xml`

MAY:
- Tune NCCL buffer sizes: `export NCCL_BUFFSIZE=8388608`

## Safety Ratings

```
safety_ratings:
  - "fi_info -p efa: GREEN — read-only provider check"
  - "nccl-tests: YELLOW — consumes GPU resources during testing"
  - "export NCCL_*: GREEN — sets environment variables for current session"
  - "nvidia-smi: GREEN — read-only GPU status check"
```

## Escalation Conditions

- NCCL initialization fails with OFI plugin errors despite correct installation
- GPUDirect RDMA causes GPU hangs or kernel panics
- NCCL performance is significantly below expected bandwidth
- Training jobs fail intermittently with NCCL timeout errors

## Data Sensitivity

- NCCL debug output: MEDIUM — contains network topology and GPU configuration
- `nvidia-smi` output: LOW — GPU hardware and driver information
- NCCL test results: LOW — performance numbers
- Topology files: LOW — hardware topology description

## Prohibited Actions

- NEVER set NCCL_NET=Socket as a permanent fix — this disables EFA and uses TCP
- NEVER disable GPUDirect RDMA without measuring the performance impact
- NEVER ignore NCCL version mismatches between nodes in a training cluster
- NEVER run NCCL tests on production GPU instances during active training jobs

## Phase 3 — Rollback

1. Revert NCCL environment variables to defaults: `unset NCCL_NET NCCL_DEBUG FI_PROVIDER`
2. If aws-ofi-nccl was updated, reinstall the previous version
3. If GPU drivers were modified, reboot to restore previous driver state
4. Verify basic GPU functionality: `nvidia-smi`
5. Run a small-scale NCCL test to confirm communication is restored

## Common Issues

- symptoms: "NCCL WARN NET/OFI Could not find nccl-net-ofi plugin"
  diagnosis: "aws-ofi-nccl plugin is not installed or not in the library path."
  resolution: "Install EFA with GPU support: sudo ./efa_installer.sh -y -g. Set LD_LIBRARY_PATH to include /opt/amazon/efa/lib64."

- symptoms: "NCCL INFO Using network Socket — training is very slow"
  diagnosis: "NCCL is falling back to TCP sockets instead of using EFA."
  resolution: "Set NCCL_NET=ofi and FI_PROVIDER=efa. Verify aws-ofi-nccl plugin is installed."

## Output Format

```yaml
root_cause: "nccl — <specific_cause>"
evidence:
  - type: nccl_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix NCCL/EFA configuration"
  long_term: "Standardize NCCL environment in training container images"
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
