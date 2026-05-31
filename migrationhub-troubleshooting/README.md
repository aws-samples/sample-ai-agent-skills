# Migration Hub Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Migration Hub problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Migration Hub — discovery, assessment, migration tracking, strategy recommendations, refactor spaces, and application grouping.

### Activate When

- Home region configuration issues
- Discovery data collection failures
- Assessment or strategy recommendation errors
- Migration tracking or status issues
- Refactor spaces environment problems
- Application grouping errors

---

## Skill Structure

```
migrationhub-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-data-collection.md
    ├── A2-data-import.md
    ├── B1-portfolio-assessment.md
    ├── B2-strategy-assessment.md
    ├── C1-migration-tasks.md
    ├── C2-progress-tracking.md
    ├── C3-status-updates.md
    ├── D1-strategy-recommendations.md
    ├── D2-analysis-config.md
    ├── E1-refactor-spaces.md
    ├── E2-routing-config.md
    ├── F1-application-grouping.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `mgh:*`, `migrationhub-strategy:*`, `migration-hub-refactor-spaces:*`, `discovery:*`
- Home region configured

---

## License

MIT-0
