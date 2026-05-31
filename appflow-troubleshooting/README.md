# Amazon AppFlow Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon AppFlow problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon AppFlow — flow creation, connector profiles, flow execution, data mapping, scheduling, destinations, custom connectors, private connections, and data encryption.

### Activate When

- Flow creation failures
- Connector profile errors
- Salesforce connector issues
- Flow execution failures
- Data mapping errors
- Scheduling issues
- S3 or Redshift destination errors
- Custom connector issues
- Private connection failures
- Data encryption problems
- General AppFlow errors

---

## Skill Structure

```
appflow-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-flow-creation-failures.md
    ├── A2-flow-configuration.md
    ├── B1-connector-profile-errors.md
    ├── B2-salesforce-connector.md
    ├── C1-flow-execution-failures.md
    ├── C2-data-mapping-errors.md
    ├── D1-scheduling-issues.md
    ├── D2-trigger-configuration.md
    ├── E1-s3-destination-errors.md
    ├── E2-redshift-destination-errors.md
    ├── F1-custom-connector-issues.md
    ├── F2-private-connection.md
    ├── G1-data-encryption.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Flow Management** | A1–A2 | Creation failures, configuration |
| **B — Connectors** | B1–B2 | Profile errors, Salesforce connector |
| **C — Execution** | C1–C2 | Execution failures, data mapping |
| **D — Scheduling** | D1–D2 | Scheduling issues, trigger configuration |
| **E — Destinations** | E1–E2 | S3 destination, Redshift destination |
| **F — Advanced** | F1–F2 | Custom connectors, private connections |
| **G — Security** | G1 | Data encryption |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `appflow:*`, `s3:Get*`, `redshift:Describe*`, `kms:Describe*`, `iam:Get*`, `cloudwatch:GetMetricStatistics` permissions

---

## License

MIT-0
