# Elastic Fabric Adapter Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EFA issue.

## Guardrail 1: EFA Is Instance-Type Specific
EFA is only supported on specific instance types (p4d, p5, trn1, hpc6a, hpc7g, c5n.18xlarge, c6in, m5n, r5n, etc.). Not all large or compute-optimized instances support EFA. Always verify with `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].NetworkInfo.EfaSupported'`.

## Guardrail 2: Security Group Must Allow All Self-Traffic
EFA requires a security group with an inbound rule allowing ALL traffic from itself (self-referencing security group). Without this rule, EFA OS-bypass traffic is blocked. Standard TCP/UDP rules are insufficient.

## Guardrail 3: EFA Driver Must Be Installed Separately
The EFA driver and libfabric are NOT included in standard Amazon Linux or Ubuntu AMIs. They must be installed using the EFA installer package. Without the driver, the interface functions as a standard ENA adapter only.

## Guardrail 4: EFA Cannot Be Hot-Attached to Running Instances
EFA interfaces must be specified at launch time or attached when the instance is stopped. Unlike standard ENIs, EFA interfaces cannot be attached to running instances. Plan EFA configuration before launch.

## Guardrail 5: Placement Groups Are Critical for Performance
Cluster placement groups ensure instances are on the same network spine, minimizing latency. Without a cluster placement group, EFA instances may be placed across different racks or availability zones, severely degrading performance.

## Guardrail 6: EFA and ENA Are Different but Related
An EFA interface also functions as an ENA interface for standard IP traffic. EFA adds OS-bypass capabilities on top of ENA. Do not confuse EFA-specific issues (OS-bypass, libfabric) with standard networking issues (ENA, TCP/IP).

## Guardrail 7: NCCL Requires aws-ofi-nccl Plugin
NVIDIA NCCL does not natively support EFA. The aws-ofi-nccl plugin bridges NCCL to libfabric/EFA. Without this plugin, NCCL falls back to TCP sockets, resulting in dramatically lower performance.

## Guardrail 8: GDR (GPU Direct RDMA) Has Additional Requirements
GPUDirect RDMA over EFA requires specific instance types (p4d, p5), the nvidia-peermem kernel module, and compatible CUDA/driver versions. Not all EFA-capable GPU instances support GDR.

## Guardrail 9: EFA Performance Depends on Message Size
EFA performance characteristics vary with message size. Small messages are latency-sensitive and benefit from OS-bypass. Large messages are bandwidth-sensitive. Benchmark with representative message sizes, not just peak bandwidth tests.

## Guardrail 10: Multiple EFA Interfaces Per Instance
Some instance types support multiple EFA interfaces (e.g., p4d supports 4, p5 supports 32). Each interface provides additional bandwidth. Multi-EFA requires proper configuration in MPI/NCCL to utilize all interfaces.

## Guardrail 11: EFA Traffic Does Not Leave the VPC
EFA OS-bypass traffic is confined to the VPC. It does not traverse VPC peering, Transit Gateway, or VPN connections. All communicating instances must be in the same VPC (or use standard TCP fallback).

## Guardrail 12: Hugepages May Be Required for Optimal Performance
EFA and libfabric benefit from hugepages for memory registration. If hugepages are not configured, performance may be suboptimal and memory registration may fail for large buffers. Configure hugepages in the OS before running EFA workloads.
