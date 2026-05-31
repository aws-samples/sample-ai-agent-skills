---
title: "H3 — VPC Endpoint Policy Issues"
description: "Diagnose access failures caused by restrictive VPC endpoint policies"
status: active
severity: MEDIUM
triggers:
  - "Access denied through endpoint"
  - "Endpoint policy blocking"
  - "S3 access denied via gateway endpoint"
owner: devops-agent
objective: "Fix VPC endpoint policy to allow required access"
context: "Endpoint policies are resource-based policies that control which AWS resources can be accessed through the endpoint. Default policy allows full access. Restrictive policies can block legitimate traffic. Endpoint policy AND IAM policy must BOTH allow the action."
---

## Phase 1 — Triage

MUST:
- Get the endpoint policy: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>` → PolicyDocument
- Check if the policy allows the required actions, resources, and principals
- Verify IAM policy also allows the action (both must allow)
- Check if the default full-access policy was replaced with a restrictive one

SHOULD:
- Test with the default full-access policy to confirm endpoint policy is the issue
- Check if S3 bucket policy has a VPC endpoint condition that might conflict
- Verify the principal in the endpoint policy matches the calling role/user

MAY:
- Check CloudTrail for access denied events referencing the endpoint
- Review if a compliance automation applied the restrictive policy

## Common Issues

- symptoms: "Can access some S3 buckets but not others through gateway endpoint"
  diagnosis: "Endpoint policy restricts access to specific bucket ARNs."
  resolution: "Add the required bucket ARN to the endpoint policy Resource list."

- symptoms: "Access denied after endpoint policy update"
  diagnosis: "New endpoint policy is too restrictive or has syntax errors."
  resolution: "Fix the endpoint policy or temporarily revert to default full-access policy."

## Safety Ratings

```
safety_ratings:
  - "describe-vpc-endpoints: GREEN — read-only endpoint and policy inspection"
  - "describe-flow-logs: GREEN — read-only flow log query"
  - "modify-vpc-endpoint (update endpoint policy): YELLOW — changes policy, recoverable by reverting to previous policy"
  - "modify-vpc-endpoint (reset to default full-access policy): YELLOW — broadens access, recoverable by re-applying restrictive policy"
```

## Escalation Conditions

- "Fix requires modifying endpoint policy in production"
- "Fix requires broadening endpoint policy — may conflict with compliance requirements"
- "Multiple services affected by restrictive endpoint policy"
- "Fix involves S3 bucket policy conditions referencing the endpoint"

## Data Sensitivity

- HIGH: endpoint policy (exposes which AWS resources are accessible through the endpoint)
- HIGH: S3 bucket policies with VPC endpoint conditions (expose access control architecture)
- MEDIUM: IAM policy details, CloudTrail access denied events

## Prohibited Actions

- "NEVER suggest 0.0.0.0/0 inbound on security groups"
- "NEVER suggest removing all NACL rules to test connectivity"
- "NEVER suggest disabling VPC flow logs"
- "NEVER suggest deleting a VPC without confirming all resources are removed"
- "NEVER set endpoint policy to full access in production without compliance review"

## Phase 3 — Rollback

- If endpoint policy was updated: revert it with `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <vpce-id> --policy-document <original-policy-json>`
- If endpoint policy was reset to default: re-apply the restrictive policy with `aws ec2 modify-vpc-endpoint --vpc-endpoint-id <vpce-id> --policy-document <restrictive-policy-json>`
- Document the original endpoint policy document before making changes

## Output Format

```yaml
root_cause: "endpoint_policy — <detail>"
evidence:
  - type: vpc_endpoint
    content: "<endpoint policy document and denied action>"
severity: MEDIUM
mitigation:
  immediate: "Update endpoint policy to allow required access"
  long_term: "Use broad endpoint policies with IAM for fine-grained control"
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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
