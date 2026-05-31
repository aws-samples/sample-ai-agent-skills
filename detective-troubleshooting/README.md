# Detective Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Detective problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Detective when the console alone isn't enough — behavior graph creation and configuration, data source enablement, finding investigation workflows, entity profiles, anomaly detection, multi-account management, GuardDuty and Security Hub integration, data volume monitoring, retention, and access control.

### Activate When

- Behavior graph creation fails
- Data sources not enabled or not ingesting
- Finding investigation not producing results
- Entity profiles missing or incomplete
- Anomaly detection not working
- Admin account configuration issues
- Member invitation failures or pending
- GuardDuty findings not appearing in Detective
- Security Hub integration problems
- Data volume unexpectedly high or low
- Data retention questions
- IAM permission errors for Detective operations
- Cross-account investigation access issues

---

## Skill Structure

```
detective-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-behavior-graph.md
    ├── A2-data-sources.md
    ├── B1-finding-investigation.md
    ├── B2-entity-profiles.md
    ├── B3-anomalies.md
    ├── C1-admin-account.md
    ├── C2-member-invitations.md
    ├── D1-guardduty-findings.md
    ├── D2-security-hub.md
    ├── E1-data-volume.md
    ├── E2-retention.md
    ├── F1-iam-permissions.md
    ├── F2-cross-account.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (18 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Graph** | A1–A2 | Behavior graph issues, data sources |
| **B — Investigation** | B1–B3 | Finding investigation, entity profiles, anomalies |
| **C — Multi-Account** | C1–C2 | Admin account, member invitations |
| **D — Integration** | D1–D2 | GuardDuty findings, Security Hub |
| **E — Data** | E1–E2 | Data volume, retention |
| **F — Access** | F1–F2 | IAM permissions, cross-account |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
