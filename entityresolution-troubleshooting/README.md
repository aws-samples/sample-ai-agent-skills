# AWS Entity Resolution Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Entity Resolution problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Entity Resolution — schema mapping, matching workflows (rule-based and ML), ID mapping workflows, provider service integration, output configuration, and job execution issues.

### Activate When

- Schema mapping creation or validation failures
- Matching workflow configuration errors
- Rule-based matching not producing expected results
- ML matching quality issues
- ID mapping workflow failures
- Provider service integration problems
- Output configuration or S3 access issues
- Job execution failures or timeouts
- General Entity Resolution errors

---

## Skill Structure

```
entityresolution-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-schema-mapping-creation.md
    ├── A2-schema-mapping-validation.md
    ├── B1-matching-workflow.md
    ├── B2-rule-based-matching.md
    ├── C1-ml-matching-config.md
    ├── C2-match-quality.md
    ├── D1-id-mapping-workflow.md
    ├── D2-provider-services.md
    ├── E1-output-configuration.md
    ├── E2-output-analysis.md
    ├── F1-job-execution.md
    ├── F2-job-failures.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `entityresolution:*`, `glue:GetTable`, `s3:*`, `cloudtrail:LookupEvents`
- Glue Data Catalog tables configured for input data
- Provider service subscriptions for ID mapping (if applicable)

---

## License

MIT-0
