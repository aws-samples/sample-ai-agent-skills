# Amazon Kendra Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Kendra problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Kendra — index creation, data source sync, query relevance, FAQ management, custom document enrichment, access control, and experience builder.

### Activate When

- Index creation failures or slow provisioning
- Data source sync errors or incomplete sync
- Poor query relevance or missing results
- FAQ import failures
- Custom document enrichment errors
- Access control / ACL issues
- Experience builder configuration problems
- General Kendra errors without clear symptoms

---

## Skill Structure

```
kendra-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-index-creation.md
    ├── A2-index-configuration.md
    ├── B1-sync-failures.md
    ├── B2-connector-config.md
    ├── B3-crawl-issues.md
    ├── C1-query-relevance.md
    ├── C2-query-suggestions.md
    ├── C3-query-filtering.md
    ├── D1-faq-import.md
    ├── D2-faq-quality.md
    ├── E1-custom-enrichment.md
    ├── E2-enrichment-lambda.md
    ├── F1-acl-configuration.md
    ├── F2-token-access.md
    ├── G1-experience-builder.md
    ├── G2-sso-integration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with Kendra, S3, IAM, CloudWatch, Lambda permissions
- IAM Identity Center for experience builder features

---

## License

MIT-0
