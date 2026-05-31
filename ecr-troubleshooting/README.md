# ECR Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Elastic Container Registry (ECR) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for ECR when the console alone isn't enough — Docker push/pull failures, authentication token expiry, image scanning issues, lifecycle policy misconfigurations, tag immutability conflicts, repository creation and cross-account access, cross-region and cross-account replication, repository policies, encryption, pull throughput, caching, and ECR Public gallery issues.

### Activate When

- Docker push fails with authorization or denied errors
- Docker pull fails with manifest not found or access denied
- ECR authentication token expired or invalid
- Image vulnerability scanning not completing or showing unexpected results
- Lifecycle policy not cleaning up images as expected
- Tag immutability preventing image push
- Repository creation failures
- Cross-account image pull/push denied
- Cross-region replication not working or lagging
- Cross-account replication failures
- Repository policy misconfiguration
- KMS encryption errors on push/pull
- Pull throughput bottlenecks or rate limiting
- Pull-through cache not working
- ECR Public repository issues
- ECR Public gallery visibility problems

---

## Skill Structure

```
ecr-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-push-failures.md
    ├── A2-pull-failures.md
    ├── A3-auth-token.md
    ├── B1-image-scanning.md
    ├── B2-lifecycle-policies.md
    ├── B3-tag-immutability.md
    ├── C1-repository-creation.md
    ├── C2-cross-account-access.md
    ├── D1-cross-region-replication.md
    ├── D2-cross-account-replication.md
    ├── E1-repository-policy.md
    ├── E2-encryption.md
    ├── F1-pull-throughput.md
    ├── F2-caching.md
    ├── G1-ecr-public.md
    ├── G2-gallery-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Push/Pull** | A1–A3 | Push failures, pull failures, auth token issues |
| **B — Images** | B1–B3 | Image scanning, lifecycle policies, tag immutability |
| **C — Repository** | C1–C2 | Creation failures, cross-account access |
| **D — Replication** | D1–D2 | Cross-region replication, cross-account replication |
| **E — Security** | E1–E2 | Repository policy, encryption |
| **F — Performance** | F1–F2 | Pull throughput, caching |
| **G — Public** | G1–G2 | ECR Public, gallery issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
