# SMS Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Server Migration Service (SMS) problems using structured runbooks.

---

## What This Skill Covers

Structured troubleshooting for AWS SMS — replication jobs, server catalog, AMI generation, application migration, multi-server coordination, and scheduling.

**Note:** SMS is legacy. AWS recommends Application Migration Service (MGN) for new migrations.

### Activate When

- Replication job or run failures
- Server catalog or connector issues
- AMI generation errors
- Application migration problems
- Multi-server coordination issues
- Replication scheduling errors

---

## Skill Structure

```
sms-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-replication-jobs.md, A2-run-failures.md
    ├── B1-server-catalog.md, B2-connector.md
    ├── C1-ami-generation.md, C2-launch-config.md
    ├── D1-app-migration.md, D2-server-groups.md
    ├── E1-multi-server.md, F1-scheduling.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md, hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `sms:*`, `ec2:*`, `s3:*`, `iam:GetRole`
- SMS Connector deployed in source environment

---

## License

MIT-0
