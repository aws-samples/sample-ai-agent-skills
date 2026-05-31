---
title: "A2 — Container Policy Errors"
description: "Diagnose MediaStore container policy configuration errors"
status: active
severity: HIGH
triggers: ["policy error", "access denied", "container policy failed", "permission denied mediastore"]
owner: devops-agent
objective: "Resolve MediaStore container policy configuration errors and access issues"
context: "Container policies control access to MediaStore containers. They use MediaStore-specific actions, not S3 actions."
---

## Phase 1 — Triage

MUST:
- Get current container policy: `aws mediastore get-container-policy --container-name <name>`
- Verify the policy uses MediaStore actions (mediastore:GetObject, not s3:GetObject)
- Check IAM caller permissions for mediastore:PutContainerPolicy

SHOULD:
- Validate policy JSON syntax before applying
- Check if the policy Principal matches the intended IAM entities

MAY:
- Review CloudTrail for PutContainerPolicy errors
- Compare with a known-working container policy

## Phase 2 — Remediate

MUST:
- Use correct MediaStore actions: `mediastore:GetObject`, `mediastore:PutObject`, `mediastore:DeleteObject`, `mediastore:DescribeObject`, `mediastore:ListItems`
- Set the policy: `aws mediastore put-container-policy --container-name <name> --policy file://policy.json`
- Verify the policy was applied: `aws mediastore get-container-policy --container-name <name>`

SHOULD:
- Use conditions to restrict access by IP, VPC endpoint, or time
- Include explicit deny statements for sensitive operations

MAY:
- Use AWS Policy Simulator to test policy effectiveness

## Safety Ratings

```
safety_ratings:
  - "mediastore get-container-policy: GREEN — read-only policy retrieval"
  - "mediastore put-container-policy: YELLOW — changes access control"
  - "mediastore delete-container-policy: RED — removes all access restrictions"
  - "cloudtrail lookup-events: GREEN — read-only event query"
```

## Escalation Conditions

- Policy changes lock out all access including the administrator
- Policy syntax is valid but access is still denied (IAM vs resource policy conflict)
- Container policy needs to allow CloudFront access without making content public
- Organization SCP conflicts with the container policy

## Data Sensitivity

- Container policy document: MEDIUM — reveals access control configuration
- IAM principal ARNs in policy: MEDIUM — reveals account structure
- CloudTrail policy events: MEDIUM — reveals who changed policies and when
- Container endpoint: LOW — required for data plane access

## Prohibited Actions

- NEVER delete the container policy without having a replacement ready
- NEVER use Principal "*" without restrictive conditions
- NEVER use S3 action names (s3:GetObject) in MediaStore policies
- NEVER apply untested policies to production containers

## Phase 3 — Rollback

1. Restore previous policy from backup: `aws mediastore put-container-policy --container-name <name> --policy file://previous-policy.json`
2. If policy was deleted, re-apply a restrictive default policy
3. If locked out, use the account root user or IAM admin to restore access
4. Verify access is restored for intended principals
5. Document the policy change and its impact

## Common Issues

- symptoms: "PutContainerPolicy returns MalformedPolicyDocumentException"
  diagnosis: "Policy JSON is malformed or uses S3 action names instead of MediaStore actions."
  resolution: "Validate JSON syntax. Replace s3:GetObject with mediastore:GetObject and similar."

- symptoms: "Access denied despite container policy allowing the principal"
  diagnosis: "IAM policy on the caller does not include mediastore-data permissions."
  resolution: "Add mediastore-data:GetObject (data plane) permissions to the IAM policy in addition to the container policy."

## Output Format

```yaml
root_cause: "container_policy — <specific_cause>"
evidence:
  - type: policy_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix policy syntax or actions"
  long_term: "Version control container policies in IaC"
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
