---
name: servicecatalog-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Service Catalog
  problems by analyzing portfolio management, product provisioning,
  constraints, provisioned products, governance, organization sharing,
  and following structured runbooks.
  Activate when: portfolio creation failures, product provisioning errors,
  launch constraint issues, template constraint problems, provisioned
  product failures, update errors, tag option issues, budget integration
  problems, organization sharing failures, StackSet constraint errors,
  Terraform product issues, or the user says something is wrong with
  Service Catalog.
compatibility: >
  Requires AWS CLI or SDK access with servicecatalog, iam,
  cloudformation, organizations, budgets, and cloudtrail permissions.
---

# AWS Service Catalog Diagnostics

## When to use

Any AWS Service Catalog investigation — portfolio management, product provisioning, constraints, provisioned products, governance, organization sharing, StackSet constraints, or Terraform product configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws servicecatalog list-portfolios --query 'PortfolioDetails[*].{Id:Id,Name:DisplayName,Provider:ProviderName}'
aws servicecatalog search-products-as-admin --query 'ProductViewDetails[*].ProductViewSummary.{Id:ProductId,Name:Name,Type:Type}'
aws servicecatalog scan-provisioned-products --query 'ProvisionedProducts[*].{Id:Id,Name:Name,Status:Status}'
```

### Step 2 — Domain deep dive

```
aws servicecatalog describe-portfolio --id <portfolio-id>
aws servicecatalog describe-product-as-admin --id <product-id>
aws servicecatalog describe-provisioned-product --id <provisioned-product-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicecatalog.amazonaws.com --max-results 20
aws servicecatalog list-constraints-for-portfolio --portfolio-id <portfolio-id>
aws servicecatalog describe-record --id <record-id> --query 'RecordDetail.{Status:Status,RecordErrors:RecordErrors}'
```

Read `references/guardrails.md` before concluding on any Service Catalog issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `servicecatalog list-portfolios` | List all portfolios |
| `servicecatalog describe-portfolio` | Get portfolio details |
| `servicecatalog search-products-as-admin` | List all products |
| `servicecatalog describe-product-as-admin` | Get product details |
| `servicecatalog scan-provisioned-products` | List provisioned products |
| `servicecatalog describe-record` | Get provisioning record details |
| `servicecatalog list-constraints-for-portfolio` | List constraints |

## Gotchas: AWS Service Catalog

- Service Catalog has TWO personas: administrators (manage portfolios/products) and end users (provision products). Permissions differ significantly.
- Launch constraints specify the IAM role used to provision the product. Without a launch constraint, the end user's permissions are used, which often fails.
- Template constraints restrict CloudFormation parameter values. They use JSON rules syntax, not IAM policy syntax.
- Provisioned products are CloudFormation stacks under the hood. Stack failures are the most common provisioning issue.
- Portfolio sharing can be done via account ID, organization, or organizational unit. Organization sharing requires Organizations integration.
- Tag options are different from resource tags. Tag options enforce specific tag key-value pairs during provisioning.
- Terraform products use the Terraform Cloud/Enterprise engine. They require a Terraform Cloud workspace connection.

## Anti-hallucination rules

1. Always cite specific portfolio IDs, product IDs, or record IDs as evidence.
2. Launch constraints and template constraints are DIFFERENT. Never conflate them.
3. End user permissions vs admin permissions are separate. Never mix them up.
4. Provisioned products are CloudFormation stacks. Always check stack events for failures.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Portfolios | A1-A2 | Portfolio creation, portfolio sharing |
| B — Products | B1-B2 | Product provisioning failures, product version issues |
| C — Constraints | C1-C2 | Launch constraint errors, template constraint issues |
| D — Provisioned Products | D1-D2 | Provisioned product failures, update failures |
| E — Governance | E1-E2 | Tag option issues, budget integration |
| F — Organization | F1-F2 | Organization sharing, StackSet constraints |
| G — Terraform | G1 | Terraform products |
| Z — Catch-All | Z1 | General troubleshooting |
