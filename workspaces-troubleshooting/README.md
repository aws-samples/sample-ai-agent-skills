# WorkSpaces Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon WorkSpaces problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon WorkSpaces when the console alone isn't enough — launch failures, bundle issues, directory integration, client connectivity, streaming protocol problems, MFA configuration, latency and bandwidth issues, AD Connector and Managed AD problems, VPC networking, rebuild/restore operations, encryption, and access control.

### Activate When

- WorkSpace fails to launch or stays in PENDING state
- Bundle creation or custom image issues
- Directory registration or integration failures
- Client cannot connect to WorkSpace
- PCoIP or WSP streaming protocol errors
- MFA/RADIUS authentication failures
- High latency or poor streaming quality
- Bandwidth throttling or network congestion
- AD Connector connectivity issues
- AWS Managed Microsoft AD problems
- VPC subnet or security group misconfiguration
- WorkSpace cannot access internet
- Rebuild or restore operations failing
- Maintenance window issues
- Volume encryption problems
- IP access control group blocking connections

---

## Skill Structure

```
workspaces-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-launch-failures.md
    ├── A2-bundle-issues.md
    ├── A3-directory-integration.md
    ├── B1-client-connection.md
    ├── B2-streaming-protocol.md
    ├── B3-mfa-issues.md
    ├── C1-latency.md
    ├── C2-bandwidth.md
    ├── D1-ad-connector.md
    ├── D2-managed-ad.md
    ├── E1-vpc-config.md
    ├── E2-internet-access.md
    ├── F1-rebuild-restore.md
    ├── F2-maintenance.md
    ├── G1-encryption.md
    ├── G2-access-control.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (22 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Provisioning** | A1–A3 | Launch failures, bundle issues, directory integration |
| **B — Connectivity** | B1–B3 | Client connection, streaming protocol, MFA |
| **C — Performance** | C1–C2 | Latency, bandwidth |
| **D — Directory** | D1–D2 | AD Connector, AWS Managed AD |
| **E — Networking** | E1–E2 | VPC config, internet access |
| **F — Management** | F1–F2 | Rebuild, restore, maintenance |
| **G — Security** | G1–G2 | Encryption, access control |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
