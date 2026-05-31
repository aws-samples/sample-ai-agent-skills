# AWS Service Catalog Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Service Catalog problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Service Catalog — portfolio management, product provisioning, launch constraints, template constraints, provisioned products, tag options, budget integration, organization sharing, StackSet constraints, and Terraform products.

### Activate When

- Portfolio creation or sharing failures
- Product provisioning errors
- Product version issues
- Launch constraint errors
- Template constraint issues
- Provisioned product failures
- Update failures
- Tag option issues
- Budget integration problems
- Organization sharing errors
- StackSet constraint issues
- Terraform product problems
- General Service Catalog errors

---

## Skill Structure

```
servicecatalog-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-portfolio-creation.md
    ├── A2-portfolio-sharing.md
    ├── B1-product-provisioning-failures.md
    ├── B2-product-version-issues.md
    ├── C1-launch-constraint-errors.md
    ├── C2-template-constraint-issues.md
    ├── D1-provisioned-product-failures.md
    ├── D2-update-failures.md
    ├── E1-tag-option-issues.md
    ├── E2-budget-integration.md
    ├── F1-organization-sharing.md
    ├── F2-stackset-constraints.md
    ├── G1-terraform-products.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Portfolios** | A1–A2 | Creation, sharing |
| **B — Products** | B1–B2 | Provisioning failures, version issues |
| **C — Constraints** | C1–C2 | Launch constraints, template constraints |
| **D — Provisioned Products** | D1–D2 | Failures, update failures |
| **E — Governance** | E1–E2 | Tag options, budget integration |
| **F — Organization** | F1–F2 | Organization sharing, StackSet constraints |
| **G — Terraform** | G1 | Terraform products |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `servicecatalog:*`, `iam:Get*`, `cloudformation:Describe*`, `organizations:Describe*` permissions

---

## License

MIT-0
