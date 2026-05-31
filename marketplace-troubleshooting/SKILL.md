---
name: marketplace-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Marketplace
  problems by analyzing subscription failures, procurement errors,
  SaaS integrations, AMI/container product launches, metering,
  entitlements, seller registration, private marketplace, and
  following structured runbooks.
  Activate when: subscription failures, procurement errors, SaaS
  integration issues, AMI launch errors, container product problems,
  data product access issues, seller registration errors, product
  listing issues, metering failures, entitlement errors, private
  marketplace configuration, organization procurement, vendor
  insights, or the user says something is wrong with Marketplace.
compatibility: >
  Requires AWS CLI or SDK access with aws-marketplace, aws-marketplace-catalog,
  aws-marketplace-entitlement, aws-marketplace-metering, organizations,
  iam, cloudtrail, and cloudwatch permissions.
---

# AWS Marketplace Diagnostics

## When to use

Any AWS Marketplace investigation — subscription management, procurement workflows, SaaS integration, AMI/container product launches, data product access, seller operations, metering, entitlements, private marketplace, or organization procurement.

## Investigation workflow

### Step 1 — Collect and triage

```
aws marketplace-catalog list-entities --catalog AWSMarketplace --entity-type Offer --query 'EntitySummaryList[*].{Id:EntityId,Name:Name,Type:EntityType}'
aws marketplace-entitlement get-entitlements --product-code <product-code> --query 'Entitlements[*].{Dimension:Dimension,Value:Value,ExpirationDate:ExpirationDate}'
aws ce get-cost-and-usage --time-period Start=2025-03-01,End=2025-04-01 --granularity MONTHLY --filter '{"Dimensions":{"Key":"LEGAL_ENTITY_NAME","Values":["AWS Marketplace"]}}' --metrics UnblendedCost
```

### Step 2 — Domain deep dive

```
aws marketplace-catalog describe-entity --catalog AWSMarketplace --entity-id <entity-id>
aws marketplace-metering batch-meter-usage --product-code <code> --usage-records '[{"Timestamp":"2025-04-12T00:00:00Z","CustomerIdentifier":"<id>","Dimension":"<dim>","Quantity":0}]' --dry-run
aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==`aws-marketplace.amazonaws.com`]'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aws-marketplace.amazonaws.com --max-results 20
aws marketplace-catalog list-change-sets --catalog AWSMarketplace --filter-list '[{"Name":"EntityId","ValueList":["<entity-id>"]}]'
aws servicecatalog search-provisioned-products --filters Key=SearchQuery,Value=["marketplace"]
```

Read `references/guardrails.md` before concluding on any Marketplace issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `marketplace-catalog list-entities` | List Marketplace entities |
| `marketplace-catalog describe-entity` | Get entity details |
| `marketplace-entitlement get-entitlements` | Check entitlements |
| `marketplace-metering batch-meter-usage` | Test metering records |
| `marketplace-catalog start-change-set` | Initiate catalog changes |
| `organizations list-aws-service-access` | Check org integration |
| `servicecatalog search-provisioned-products` | Find provisioned products |

## Gotchas: AWS Marketplace

- Marketplace subscriptions are ACCOUNT-level. Each AWS account must subscribe independently unless using organization procurement with AWS Organizations.
- SaaS products require the buyer to complete registration on the seller's website AFTER subscribing. The subscription alone does not provision access.
- AMI products launch EC2 instances. The buyer pays both the software price AND the underlying EC2 infrastructure costs separately.
- Container products may require ECS or EKS cluster setup before deployment. The Marketplace subscription only provides access to the container image.
- Metering records must be sent within 6 hours of usage. Late records are rejected. Sellers must implement retry logic for failed metering calls.
- Private Marketplace is an organization-level feature that restricts which products accounts can subscribe to. It requires AWS Organizations.
- Product codes are immutable identifiers. They do not change even if the product name or seller changes.

## Anti-hallucination rules

1. Always cite specific product codes, entity IDs, or API responses as evidence.
2. Marketplace subscriptions and entitlements are separate concepts. Subscriptions grant access; entitlements define usage limits.
3. SaaS, AMI, container, and data products have different fulfillment mechanisms. Never conflate them.
4. Private Marketplace restricts procurement — it does NOT manage existing subscriptions.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Subscriptions | A1-A2 | Subscription failures, procurement errors |
| B — Product Launch | B1-B2 | SaaS integration issues, AMI product launch errors |
| C — Containers & Data | C1-C2 | Container product issues, data product access |
| D — Seller Operations | D1-D2 | Seller registration errors, product listing issues |
| E — Metering & Entitlements | E1-E2 | Metering failures, entitlement errors |
| F — Organization | F1-F2 | Private marketplace config, organization procurement |
| G — Insights | G1 | Vendor insights |
| Z — Catch-All | Z1 | General troubleshooting |
