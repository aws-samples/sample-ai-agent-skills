---
title: "F1 — Cross-Account Delivery"
description: "Diagnose and resolve cross-account event and message delivery issues for SQS, SNS, and EventBridge"
status: active
severity: HIGH
triggers:
  - "cross-account"
  - "resource policy"
  - "cross-account SNS"
  - "cross-account SQS"
  - "cross-account EventBridge"
owner: devops-agent
objective: "Resolve cross-account delivery failures for event-driven services"
context: "Cross-account delivery requires resource-based policies on the receiving service (SQS queue policy, SNS topic policy, EventBridge bus policy) to allow the sending account. For EventBridge, the source account must also have a rule that targets the destination event bus. Cross-account delivery can fail due to missing policies, incorrect principal ARNs, condition key mismatches, or KMS encryption key access issues."
---

## Phase 1 — Triage

MUST:
- Check the resource policy on the receiving resource:
  - SQS: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names Policy`
  - SNS: `aws sns get-topic-attributes --topic-arn <topic-arn> --query 'Attributes.Policy'`
  - EventBridge: `aws events describe-event-bus --name <bus-name> --query 'Policy'`
- Verify the sending account's principal ARN is allowed in the policy
- Check CloudTrail in both accounts for API errors
- Verify the resource ARNs are correct (account ID, region, resource name)
- Check if the resource is encrypted with KMS and the sending account has decrypt permissions

SHOULD:
- Verify condition keys in the policy (aws:SourceAccount, aws:SourceArn) are correct
- Check if an SCP (Service Control Policy) in either account blocks cross-account access
- For EventBridge: verify the source account has a rule targeting the destination bus
- Check if VPC endpoints with restrictive policies are blocking cross-account access

MAY:
- Review AWS Organizations policies for cross-account restrictions
- Check if the cross-account setup was recently changed
- Verify DNS resolution works for cross-account endpoints

## Phase 2 — Remediate

MUST:
- If SQS policy missing: add cross-account send permission:
  ```
  aws sqs set-queue-attributes --queue-url <queue-url> --attributes '{"Policy":"{\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"<source-account-arn>\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"<queue-arn>\"}]}"}'
  ```
- If SNS policy missing: add cross-account publish permission
- If EventBridge bus policy missing: add cross-account PutEvents permission
- If KMS key blocks access: add the source account to the KMS key policy

SHOULD:
- Use condition keys to restrict cross-account access to specific resources
- Verify delivery works after policy changes by sending a test message
- Document the cross-account architecture and required policies

MAY:
- Implement AWS Organizations resource sharing for simplified cross-account access
- Set up monitoring for cross-account delivery failures
- Create IaC templates for cross-account policy management

## Common Issues

- symptoms: "SNS publish to cross-account SQS returns AccessDenied"
  diagnosis: "SQS queue policy does not allow the SNS topic from the other account."
  resolution: "Add a policy statement allowing sns:SendMessage from the source topic ARN."

- symptoms: "EventBridge PutEvents to cross-account bus fails"
  diagnosis: "Destination event bus policy does not allow the source account."
  resolution: "Add events:PutEvents permission for the source account in the bus policy."

- symptoms: "Cross-account delivery works for unencrypted but not encrypted resources"
  diagnosis: "KMS key policy does not allow the source account to use the key."
  resolution: "Add kms:Decrypt and kms:GenerateDataKey permissions for the source account in the KMS key policy."

- symptoms: "Cross-account delivery stopped working after policy update"
  diagnosis: "Policy update removed or modified the cross-account permission."
  resolution: "Review the current policy and restore the cross-account permission statement."

## Output Format

```yaml
root_cause: "Cross-account delivery failure — <specific_cause>"
evidence:
  - type: resource_policy
    content: "<resource policy from receiving account>"
  - type: cloudtrail_errors
    content: "<API error events from both accounts>"
severity: HIGH
mitigation:
  immediate: "Fix resource policies to allow cross-account access"
  long_term: "Implement IaC-managed cross-account policies with monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get/describe): GREEN — read-only"
  - "Phase 2 resource policy changes: RED — affects cross-account access control"
  - "Phase 2 KMS key policy changes: RED — affects encryption key access"

## Escalation Conditions

escalation_conditions:
  - "Cross-account policy changes require approval from both account owners"
  - "KMS key policy changes affect encryption for multiple services"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "SCP or Organizations policy is blocking the access"
  - "Issue involves more than 2 accounts in a chain"

## Data Sensitivity

data_sensitivity:
  - command: "resource policies"
    sensitivity: HIGH
    contains: "Account IDs, principal ARNs, and access permissions"
  - command: "KMS key policies"
    sensitivity: HIGH
    contains: "Encryption key access control"
  - command: "CloudTrail events"
    sensitivity: HIGH
    contains: "API call details from both accounts"

## Prohibited Actions

prohibited_actions:
  - "NEVER use Principal: * in resource policies for cross-account access"
  - "NEVER share KMS key policies without redacting sensitive account details"
  - "NEVER modify resource policies in production without testing in a staging environment"
  - "NEVER bypass cross-account restrictions by copying data instead of fixing policies"

## Phase 3 — Rollback

- Before changing resource policies, save the current policy in both accounts
- If policy changes grant unintended access, revert immediately
- If KMS key policy changes break other services, restore the previous key policy
- If cross-account delivery causes unexpected message volume, add rate limiting
- Verify both accounts' services function correctly after any policy changes
