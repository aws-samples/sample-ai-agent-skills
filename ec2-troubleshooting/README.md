# EC2 Instance Diagnostics Skill

Agent skill for investigating and troubleshooting EC2 instance problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EC2 instances when the AWS Console alone isn't enough — system logs, instance screenshots, OS-level diagnostics via SSM, network configuration, storage state, boot failures, performance issues, and connectivity problems.

### Activate When

- Instance is unreachable or unresponsive
- Status checks are failing (system or instance)
- Instance is stuck in pending/stopping/shutting-down
- SSH/RDP connections fail
- Performance is degraded (CPU, memory, disk, network)
- EBS volumes won't attach or mount
- Instance store data is lost
- Networking issues (ENI, security group, NACL, VPC routing, DNS)
- IAM/IMDS credential failures
- Launch failures (InsufficientCapacity, quota exceeded, AMI issues)
- Boot or initialization failures (kernel panic, fsck, cloud-init, GRUB)
- Unexpected termination or state transitions

---

## Skill Structure

```
ec2-troubleshooting/
├── SKILL.md                          # Main skill definition and investigation workflow
├── README.md                         # This file
└── references/
    ├── A1-insufficient-instance-capacity.md
    ├── A2-service-limit-exceeded.md
    ├── A3-ami-launch-failures.md
    ├── A4-launch-template-errors.md
    ├── B1-system-status-check-failure.md
    ├── B2-instance-status-check-failure.md
    ├── B3-ebs-volume-status-check.md
    ├── C1-ssh-rdp-connection-failure.md
    ├── C2-security-group-issues.md
    ├── C3-nacl-issues.md
    ├── C4-routing-issues.md
    ├── C5-dns-resolution-failures.md
    ├── D1-cpu-exhaustion.md
    ├── D2-memory-pressure.md
    ├── D3-disk-io-bottleneck.md
    ├── D4-network-throughput-issues.md
    ├── D5-cpu-credit-depletion.md
    ├── E1-ebs-attach-detach-failures.md
    ├── E2-volume-full.md
    ├── E3-iops-throttling.md
    ├── E4-instance-store-data-loss.md
    ├── F1-kernel-panic.md
    ├── F2-filesystem-corruption.md
    ├── F3-cloud-init-failures.md
    ├── F4-grub-boot-failures.md
    ├── G1-eni-issues.md
    ├── G2-elastic-ip-issues.md
    ├── G3-nat-gateway-issues.md
    ├── G4-vpc-peering-tgw-issues.md
    ├── H1-instance-profile-issues.md
    ├── H2-imds-issues.md
    ├── H3-kms-encryption-issues.md
    ├── I1-stuck-stopping.md
    ├── I2-stuck-pending.md
    ├── I3-unexpected-termination.md
    ├── J1-nitro-xen-migration.md
    ├── J2-bare-metal-issues.md
    ├── J3-graviton-arm-issues.md
    ├── K1-scheduled-maintenance.md
    ├── K2-instance-retirement.md
    ├── K3-host-recovery.md
    ├── Z1-general-troubleshooting.md
    ├── ec2-storage-guardrails.md
    ├── ec2-networking-guardrails.md
    └── ec2-hallucination-patterns.yaml
```

---

## Runbook Library (40 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Launch Failures** | A1–A4 | InsufficientCapacity, quota/limit exceeded, AMI issues, launch template errors |
| **B — Status Checks** | B1–B3 | System status (host), instance status (OS), EBS volume status |
| **C — Connectivity** | C1–C5 | SSH/RDP failure, security groups, NACLs, VPC routing, DNS resolution |
| **D — Performance** | D1–D5 | CPU exhaustion, memory/OOM, disk I/O bottleneck, network throughput, T-series credit depletion |
| **E — Storage** | E1–E4 | EBS attach/detach, volume full, IOPS throttling, instance store data loss |
| **F — Boot/Init** | F1–F4 | Kernel panic, filesystem corruption/fsck, cloud-init/user-data, GRUB boot loader |
| **G — Networking** | G1–G4 | ENI issues, Elastic IP, NAT gateway, VPC peering/Transit Gateway |
| **H — IAM/Security** | H1–H3 | Instance profile/role, IMDS hop limit/access, KMS/encryption |
| **I — State Transitions** | I1–I3 | Stuck stopping, stuck pending, unexpected termination |
| **J — Instance Types** | J1–J3 | Nitro vs Xen migration, bare metal, Graviton/ARM architecture |
| **K — Maintenance** | K1–K3 | Scheduled maintenance, instance retirement, Dedicated Host recovery |
| **Z — Catch-All** | Z1 | General troubleshooting for unclassified issues |

---

## Guardrails & Anti-Hallucination

The skill includes three guardrail files that prevent common misdiagnosis:

### Storage Guardrails (`ec2-storage-guardrails.md`)
11 rules covering: stop+start vs reboot semantics, instance store ephemerality, gp2 vs gp3 baseline performance, EBS-optimized bandwidth limits, NVMe device name mapping, Multi-Attach filesystem requirements, volume modification cooldowns, snapshot restore performance, and KMS encryption dependencies.

### Networking Guardrails (`ec2-networking-guardrails.md`)
12 rules covering: SG stateful vs NACL stateless behavior, SG allow-only rules, source/destination check, public IP vs EIP persistence, VPC flow log limitations, MTU/jumbo frames, VPC DNS requirements, cross-AZ costs, ENI limits, peering non-transitivity, ENA requirements, and IMDS hop limits.

### Hallucination Patterns (`ec2-hallucination-patterns.yaml`)
10 patterns that LLMs commonly get wrong about EC2, including:
- Confusing reboot with stop+start (different hardware/IP/storage effects)
- Claiming security groups have deny rules (they don't)
- Claiming instance store data survives stop (it doesn't)
- Confusing system vs instance status check ownership
- Applying SG stateful behavior to NACLs
- Misquoting gp2 baseline IOPS (3 IOPS/GB, not 3000)

---

## Investigation Workflow

Each runbook follows a consistent phased structure:

### Phase 1 — Triage
Collect initial evidence using AWS APIs (`describe-instances`, `describe-instance-status`, `get-console-output`, `get-console-screenshot`) and SSM commands. Classify the failure domain.

### Phase 2 — Enrich / Remediate
Deep dive into the specific domain using targeted APIs, CloudWatch metrics, VPC Reachability Analyzer, or OS-level diagnostics via SSM.

### Phase 3 — Report
State root cause with evidence, blast radius, severity, and recommended mitigations (immediate and long-term).

### Output Format
Every runbook produces structured YAML output:
```yaml
root_cause: "<category> — <detail>"
evidence:
  - type: <source>
    content: "<specific finding>"
severity: CRITICAL | HIGH | MEDIUM
mitigation:
  immediate: "<action>"
  long_term: "<prevention>"
```

---

## Prerequisites

- AWS CLI or SDK access with EC2, CloudWatch, and SSM permissions
- For OS-level diagnostics: SSM Agent running on target instances with `AmazonSSMManagedInstanceCore` policy
- For boot diagnostics: `get-console-output` and `get-console-screenshot` permissions
- For network analysis: VPC Reachability Analyzer permissions (optional)

---

## Usage Examples

### Instance Unreachable
```
Instance i-0abc123def in us-east-1 is unreachable via SSH. Check status
checks, security groups, NACLs, and route tables. Get the system log
if status checks are failing.
```

### Performance Investigation
```
Instance i-0abc123def is running slow. It's a t3.medium. Check if CPU
credits are exhausted, look at memory and disk usage, and check EBS
volume performance metrics.
```

### Boot Failure
```
Instance i-0abc123def won't come up after a stop+start. Get the console
screenshot and system log. It was migrated from m5.large to m6i.large.
```

### Unexpected Termination
```
Instance i-0abc123def was terminated unexpectedly overnight. Check
CloudTrail for who/what terminated it and the StateReason.
```

### General Triage
```
Something is wrong with instance i-0abc123def but I'm not sure what.
Run a general investigation — check status, logs, metrics, and follow
whichever runbook matches the symptoms.
```

---

## License

MIT-0
