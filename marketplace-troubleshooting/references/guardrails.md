# AWS Marketplace Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Marketplace issue.

## Guardrail 1: Subscriptions Are Account-Level
Marketplace subscriptions are per-account. Each AWS account must subscribe independently unless organization procurement is enabled via AWS Organizations. Do not assume a subscription in one account applies to another.

## Guardrail 2: SaaS Products Require Post-Subscribe Registration
SaaS subscriptions require the buyer to complete registration on the seller's website after subscribing in Marketplace. The subscription alone does not provision access. A redirect URL is provided after subscribe.

## Guardrail 3: AMI Products Incur Dual Costs
AMI-based products charge both the software license fee AND the underlying EC2 infrastructure costs. These appear as separate line items. Do not confuse Marketplace charges with EC2 charges.

## Guardrail 4: Metering Has a 6-Hour Window
Sellers must send metering records within 6 hours of usage via batch-meter-usage or meter-usage. Records submitted after 6 hours are rejected. Implement retry logic with exponential backoff.

## Guardrail 5: Entitlements and Subscriptions Are Separate
A subscription grants access to a product. Entitlements define usage dimensions and limits within that subscription. A valid subscription can have expired or exhausted entitlements.

## Guardrail 6: Private Marketplace Is Organization-Level
Private Marketplace is an AWS Organizations feature that restricts which products member accounts can subscribe to. It does NOT manage existing subscriptions or affect already-provisioned resources.

## Guardrail 7: Product Codes Are Immutable
Product codes are permanent identifiers assigned at product creation. They do not change when the product name, description, or seller changes. Always use product codes for programmatic lookups.

## Guardrail 8: Container Products Require Cluster Infrastructure
Container products from Marketplace provide container images but require existing ECS or EKS cluster infrastructure. The subscription does not create compute resources.

## Guardrail 9: Seller Registration Requires Tax and Banking
Seller registration requires completed tax interview and valid banking information before products can be listed. Registration status can be checked in the Marketplace Management Portal.

## Guardrail 10: Catalog API Changes Are Asynchronous
Changes made via marketplace-catalog start-change-set are asynchronous. Change sets go through PREPARING, APPLYING, and SUCCEEDED/FAILED states. Do not assume immediate effect.

## Guardrail 11: Data Products Use AWS Data Exchange
Data products listed on Marketplace are fulfilled through AWS Data Exchange. Access requires Data Exchange permissions in addition to Marketplace subscription.

## Guardrail 12: Vendor Insights Requires Seller Opt-In
Vendor Insights security profiles are only available for products where the seller has opted in and provided assessment data. Not all products have Vendor Insights profiles.
