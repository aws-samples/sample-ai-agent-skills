# AWS Glue DataBrew Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Glue DataBrew problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Glue DataBrew — project creation, recipe job execution, profile jobs, dataset configuration, data quality rulesets, scheduling, output format configuration, and S3 permission issues.

### Activate When

- Project creation or session start failures
- Recipe job execution errors
- Profile job failures or incomplete results
- Dataset connection or access issues
- Data quality rule evaluation failures
- Schedule creation or trigger problems
- Output format or compression errors
- S3 permission denied during job execution
- General DataBrew errors without clear symptoms

---

## Skill Structure

```
gluedatabrew-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-project-creation.md
    ├── A2-project-configuration.md
    ├── B1-recipe-job-failures.md
    ├── B2-recipe-steps.md
    ├── C1-profile-job-failures.md
    ├── C2-data-quality-rules.md
    ├── D1-dataset-creation.md
    ├── D2-dataset-connection.md
    ├── E1-output-format.md
    ├── E2-s3-permissions.md
    ├── F1-schedule-creation.md
    ├── F2-schedule-failures.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `databrew:*`, `s3:GetObject`, `s3:PutObject`, `iam:PassRole`, `logs:FilterLogEvents`

---

## License

MIT-0
