# Mainframe Modernization Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Mainframe Modernization problems using structured runbooks.

---

## What This Skill Covers

Structured troubleshooting for AWS Mainframe Modernization — application creation, runtime environments, batch jobs, data sets, CICS/IMS, refactoring (Blu Age), and replatforming (Micro Focus).

### Activate When

- Application creation or deployment failures
- Runtime environment issues
- Batch job or JCL execution failures
- Data set import/export problems
- CICS/IMS transaction errors
- Blu Age refactoring issues
- Micro Focus replatforming problems

---

## Skill Structure

```
mainframe-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-app-creation.md, A2-deployment.md
    ├── B1-environment-creation.md, B2-environment-config.md
    ├── C1-batch-jobs.md, C2-jcl-execution.md
    ├── D1-data-sets.md, D2-data-import-export.md
    ├── E1-cics-transactions.md, E2-ims-config.md
    ├── F1-blu-age-refactoring.md
    ├── G1-micro-focus-replatform.md, G2-testing.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md, hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `m2:*`, `s3:*`, `ec2:*`, `efs:*`

---

## License

MIT-0
