# EKS Anywhere Diagnostics Skill

Agent skill for investigating and troubleshooting EKS Anywhere problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for EKS Anywhere — cluster creation, upgrades, node provisioning, machine health checks, vSphere and bare metal providers, Cilium networking, load balancers, curated packages, Flux GitOps, cluster management service, IAM authentication, and backup/restore.

### Activate When

- Cluster creation failures
- Cluster upgrade errors
- Node provisioning issues
- Machine health check failures
- vSphere provider problems
- Bare metal provider issues
- Cilium networking errors
- Load balancer failures
- Curated package errors
- Flux GitOps sync issues
- Cluster management service problems
- IAM authenticator failures
- Backup/restore issues
- General EKS Anywhere errors

---

## Skill Structure

```
eks-anywhere-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-cluster-creation-failures.md
    ├── A2-cluster-upgrade-errors.md
    ├── B1-node-provisioning.md
    ├── B2-machine-health-checks.md
    ├── C1-vsphere-provider-issues.md
    ├── C2-bare-metal-provider-issues.md
    ├── D1-networking-cilium.md
    ├── D2-load-balancer-issues.md
    ├── E1-curated-packages.md
    ├── E2-flux-gitops.md
    ├── F1-cluster-management-service.md
    ├── F2-iam-authenticator.md
    ├── G1-backup-restore.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Cluster Lifecycle** | A1–A2 | Creation failures, upgrade errors |
| **B — Node Management** | B1–B2 | Node provisioning, machine health checks |
| **C — Providers** | C1–C2 | vSphere provider, bare metal provider |
| **D — Networking** | D1–D2 | Cilium CNI, load balancer issues |
| **E — Packages & GitOps** | E1–E2 | Curated packages, Flux GitOps |
| **F — Management & Auth** | F1–F2 | Cluster management service, IAM authenticator |
| **G — Backup & Restore** | G1 | Backup and restore |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- eksctl-anywhere CLI installed
- kubectl with access to management and workload clusters
- Provider credentials (vSphere vCenter, bare metal BMC/IPMI, CloudStack, or Nutanix)
- AWS CLI v2 for curated packages (ecr:GetAuthorizationToken)

---

## License

MIT-0
