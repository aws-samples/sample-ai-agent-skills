---
title: "D1 — Seller Registration Errors"
description: "Diagnose seller registration and onboarding issues"
status: active
severity: HIGH
triggers: ["seller registration failed", "cannot register as seller", "seller onboarding", "marketplace seller"]
owner: devops-agent
objective: "Resolve seller registration and onboarding issues"
context: "Seller registration requires tax interview completion, valid banking information, and account verification. Registration is done through the Marketplace Management Portal."
---

## Phase 1 — Triage

MUST:
- Verify account is in good standing: check Billing console
- Check seller registration status in Marketplace Management Portal
- Verify tax interview completion status

SHOULD:
- Review CloudTrail for registration-related API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=aws-marketplace.amazonaws.com --max-results 20`
- Check IAM permissions for marketplace management: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names aws-marketplace-management:viewReports`

MAY:
- Verify banking information is valid and verified
- Check if the account meets seller eligibility requirements

## Phase 2 — Remediate

MUST:
- Complete tax interview in Marketplace Management Portal
- Provide valid banking information for disbursement
- Verify account identity as required by AWS

SHOULD:
- Ensure the registering user has admin-level IAM permissions
- Complete all required seller profile fields

MAY:
- Set up a dedicated AWS account for seller operations
- Configure seller notifications for product events

## Safety Ratings

```
safety_ratings:
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "iam simulate-principal-policy: GREEN — read-only permission check"
  - "Marketplace Management Portal: YELLOW — modifies seller profile and financial settings"
  - "Tax interview submission: YELLOW — submits tax information to AWS"
```

## Escalation Conditions

- Tax interview fails validation repeatedly
- Banking information rejected after multiple attempts
- Account verification requires additional documentation
- Seller registration blocked by AWS compliance review

## Data Sensitivity

- Tax interview data: CRITICAL — contains tax identification numbers
- Banking information: CRITICAL — contains bank account details
- Seller profile: HIGH — contains business identity information
- `aws cloudtrail lookup-events`: MEDIUM — may reveal user actions

## Prohibited Actions

- NEVER log or output tax identification numbers
- NEVER store banking details in code, logs, or configuration files
- NEVER share seller registration credentials
- NEVER bypass identity verification steps

## Phase 3 — Rollback

1. If registration was started in error, contact AWS Support to cancel
2. If banking information was entered incorrectly, update through Management Portal
3. If tax interview needs correction, re-submit through the portal
4. Remove any temporary IAM permissions granted for registration
5. Document the registration attempt and status

## Common Issues

- symptoms: "Seller registration stuck in pending state"
  diagnosis: "Tax interview or banking verification incomplete."
  resolution: "Complete all required steps in Marketplace Management Portal. Check email for verification requests."

## Output Format

```yaml
root_cause: "seller_registration — <specific_cause>"
evidence:
  - type: registration_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Complete registration requirements"
  long_term: "Document seller onboarding process"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
