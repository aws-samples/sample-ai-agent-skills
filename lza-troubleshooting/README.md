# Landing Zone Accelerator on AWS (LZA) Diagnostics Skill

Agent skill for investigating and troubleshooting Landing Zone Accelerator on AWS (LZA) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Landing Zone Accelerator — installation, pipeline execution, configuration files (accounts, organization, network, security, global, customizations), CDK synthesis, CloudFormation stacks, upgrades, drift detection, and custom Config rules.

### Activate When

- LZA installation failures
- Pipeline execution errors
- Accounts configuration errors
- Organization configuration errors
- Network configuration failures
- Security configuration errors
- Global configuration issues
- Customizations configuration problems
- CDK synthesis failures
- CloudFormation stack errors
- Upgrade failures
- Drift detection issues
- Custom Config rule problems
- General LZA errors

---

## Skill Structure

```
lza-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-installation-failures.md
    ├── A2-pipeline-execution-errors.md
    ├── B1-accounts-config-errors.md
    ├── B2-organization-config-errors.md
    ├── C1-network-config-failures.md
    ├── C2-security-config-errors.md
    ├── D1-global-config-issues.md
    ├── D2-customizations-config.md
    ├── E1-cdk-synth-failures.md
    ├── E2-cloudformation-stack-errors.md
    ├── F1-upgrade-failures.md
    ├── F2-drift-detection.md
    ├── G1-custom-config-rules.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Installation/Pipeline** | A1–A2 | Installation failures, pipeline execution errors |
| **B — Account/Org Config** | B1–B2 | Accounts config errors, organization config errors |
| **C — Network/Security** | C1–C2 | Network config failures, security config errors |
| **D — Global/Custom Config** | D1–D2 | Global config issues, customizations config |
| **E — CDK/CloudFormation** | E1–E2 | CDK synth failures, CloudFormation stack errors |
| **F — Maintenance** | F1–F2 | Upgrade failures, drift detection |
| **G — Extensions** | G1 | Custom Config rules |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `codepipeline:*`, `codebuild:*`, `cloudformation:*`, `organizations:*`, `s3:Get*`, `iam:Get*`, `config:*`, `ec2:Describe*`, `cloudtrail:LookupEvents` permissions

---

## License

MIT-0
