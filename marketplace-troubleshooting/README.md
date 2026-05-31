# AWS Marketplace Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Marketplace problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Marketplace — buying, selling, and managing third-party software including subscriptions, procurement, SaaS integrations, AMI/container products, metering, entitlements, and private marketplace.

### Activate When

- Subscription failures
- Procurement errors
- SaaS integration issues
- AMI product launch errors
- Container product issues
- Data product access problems
- Seller registration errors
- Product listing issues
- Metering failures
- Entitlement errors
- Private marketplace configuration
- Organization procurement issues
- Vendor insights problems
- General Marketplace errors

---

## Skill Structure

```
marketplace-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-subscription-failures.md
    ├── A2-procurement-errors.md
    ├── B1-saas-integration-issues.md
    ├── B2-ami-product-launch-errors.md
    ├── C1-container-product-issues.md
    ├── C2-data-product-access.md
    ├── D1-seller-registration-errors.md
    ├── D2-product-listing-issues.md
    ├── E1-metering-failures.md
    ├── E2-entitlement-errors.md
    ├── F1-private-marketplace-config.md
    ├── F2-organization-procurement.md
    ├── G1-vendor-insights.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Subscriptions** | A1–A2 | Subscription failures, procurement errors |
| **B — Product Launch** | B1–B2 | SaaS integration, AMI launch errors |
| **C — Containers & Data** | C1–C2 | Container products, data product access |
| **D — Seller Operations** | D1–D2 | Seller registration, product listing |
| **E — Metering & Entitlements** | E1–E2 | Metering failures, entitlement errors |
| **F — Organization** | F1–F2 | Private marketplace, org procurement |
| **G — Insights** | G1 | Vendor insights |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `aws-marketplace:*`, `aws-marketplace-catalog:*`, `aws-marketplace-entitlement:*`, `aws-marketplace-metering:*`, `organizations:Describe*`, `iam:Get*` permissions

---

## License

MIT-0
