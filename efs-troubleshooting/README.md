# EFS Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon EFS problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EFS when the console alone isn't enough — mount failures, NFS errors, throughput bottlenecks, burst credit depletion, security group issues, access point configuration, EFS CSI driver problems, lifecycle policies, replication, and encryption.

### Activate When

- Mount command fails or times out
- NFS errors (stale file handle, permission denied)
- Throughput lower than expected
- Burst credits depleted
- Security group blocking NFS traffic
- Access point POSIX permission issues
- EFS CSI driver errors in EKS
- PersistentVolumeClaim stuck in Pending
- Lifecycle policy not transitioning files
- Replication lag or failures
- Encryption configuration issues

---

## Skill Structure

```
efs-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-mount-failures.md
    ├── A2-mount-timeout.md
    ├── A3-dns-resolution.md
    ├── B1-throughput-limits.md
    ├── B2-latency.md
    ├── B3-burst-credits.md
    ├── C1-security-group-issues.md
    ├── C2-iam-auth.md
    ├── C3-access-points.md
    ├── D1-csi-driver.md
    ├── D2-pvc-issues.md
    ├── E1-ia-transitions.md
    ├── E2-replication.md
    ├── F1-encryption-at-rest.md
    ├── F2-encryption-in-transit.md
    ├── Z1-general-troubleshooting.md
    ├── efs-guardrails.md
    └── efs-hallucination-patterns.yaml
```

---

## Runbook Library (24 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Mount Issues** | A1–A3 | Mount failures, timeout, DNS resolution |
| **B — Performance** | B1–B3 | Throughput limits, latency, burst credits |
| **C — Security** | C1–C3 | Security group issues, IAM auth, access points |
| **D — EKS Integration** | D1–D2 | CSI driver, PVC issues |
| **E — Lifecycle & Replication** | E1–E2 | IA transitions, replication |
| **F — Encryption** | F1–F2 | At-rest, in-transit |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
