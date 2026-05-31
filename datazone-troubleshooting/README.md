# DataZone Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon DataZone problems using structured runbooks.

---

## What This Skill Covers

Structured troubleshooting for Amazon DataZone — domain creation, projects, data assets, subscriptions, governance, business catalog, and data quality.

### Activate When

- Domain creation or configuration failures
- Project or environment issues
- Data asset publishing errors
- Subscription request or fulfillment problems
- Governance policy or approval workflow failures
- Business catalog or glossary issues
- Data quality or profiling errors

---

## Skill Structure

```
datazone-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-domain-creation.md, A2-domain-config.md
    ├── B1-project-management.md, B2-environments.md
    ├── C1-data-assets.md, C2-asset-publishing.md
    ├── D1-subscription-requests.md, D2-subscription-fulfillment.md
    ├── E1-governance-policies.md, E2-approval-workflows.md
    ├── F1-business-catalog.md
    ├── G1-data-quality.md, G2-data-profiling.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md, hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `datazone:*`, `glue:*`, `s3:*`, `redshift:*`, `athena:*`

---

## License

MIT-0
