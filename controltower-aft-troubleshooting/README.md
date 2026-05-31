# AWS Control Tower AFT Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Control Tower Account Factory for Terraform (AFT) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Control Tower AFT — deployment, pipeline execution, account provisioning, customizations, Terraform state, CodePipeline/CodeBuild, SSO integration, VPC configuration, and upgrades.

### Activate When

- AFT deployment failures
- AFT pipeline execution errors
- Account request failures
- Account customization errors
- Terraform state issues
- Provider configuration problems
- Global customization failures
- Account-specific customization failures
- CodePipeline errors
- CodeBuild failures
- SSO integration issues
- VPC configuration problems
- AFT upgrade issues
- General AFT errors

---

## Skill Structure

```
controltower-aft-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-aft-deployment-failures.md
    ├── A2-aft-pipeline-errors.md
    ├── B1-account-request-failures.md
    ├── B2-account-customization-errors.md
    ├── C1-terraform-state-issues.md
    ├── C2-provider-configuration.md
    ├── D1-global-customization-failures.md
    ├── D2-account-customization-failures.md
    ├── E1-codepipeline-errors.md
    ├── E2-codebuild-failures.md
    ├── F1-sso-integration.md
    ├── F2-vpc-configuration.md
    ├── G1-aft-upgrade-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Deployment** | A1–A2 | AFT deployment failures, pipeline errors |
| **B — Account Provisioning** | B1–B2 | Account request failures, customization errors |
| **C — Terraform** | C1–C2 | State issues, provider configuration |
| **D — Customizations** | D1–D2 | Global customization failures, account customization failures |
| **E — CI/CD** | E1–E2 | CodePipeline errors, CodeBuild failures |
| **F — Integration** | F1–F2 | SSO integration, VPC configuration |
| **G — Maintenance** | G1 | AFT upgrade issues |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `controltower:*`, `organizations:*`, `codepipeline:*`, `codebuild:*`, `dynamodb:*`, `s3:Get*`, `sso-admin:*`, `iam:Get*`, `lambda:Get*`, `cloudformation:Describe*` permissions

---

## License

MIT-0
