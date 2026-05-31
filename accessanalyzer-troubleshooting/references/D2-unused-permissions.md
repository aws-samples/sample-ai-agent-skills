---
title: "D2 — Unused Permissions"
description: "Diagnose unused permission finding issues"
status: active
severity: MEDIUM
triggers: ["unused permissions", "over-privileged", "excess permissions"]
owner: devops-agent
objective: "Resolve unused permission findings"
context: "Unused permission findings identify IAM policies with permissions not exercised within the tracking period."
---

## Phase 1 — Triage

MUST:
- Review unused permission findings
- Verify the tracking period covers normal usage patterns
- Check if the role/user has seasonal or infrequent usage

SHOULD:
- Cross-reference with CloudTrail for actual API usage
- Verify finding accuracy before removing permissions

MAY:
- Compare with policy generation results

## Phase 2 — Remediate

MUST:
- Verify permissions are truly unused (not seasonal/infrequent)
- Remove unused permissions from IAM policies

SHOULD:
- Use policy generation to create right-sized replacement policies
- Test permission changes in non-production first

MAY:
- Implement permission boundaries for additional safety

## Common Issues

- symptoms: "Finding shows unused permissions but role is used monthly"
  diagnosis: "Tracking period shorter than usage cycle."
  resolution: "Extend tracking period to cover full usage cycle (e.g., 180 days for monthly tasks)."

## Safety Ratings

```
safety_ratings:
  - "review unused permission findings: GREEN — read-only analysis"
  - "cross-reference with CloudTrail: GREEN — read-only audit"
  - "remove unused permissions from IAM policies: RED — changes authorization, may break workloads"
  - "implement permission boundaries: YELLOW — adds constraints, may restrict legitimate access"
```

## Escalation Conditions

- Unused permissions include high-privilege actions (iam:*, s3:*, ec2:*)
- Permission removal affects shared roles used by multiple teams or services
- Findings indicate unused admin-level permissions on production service roles
- Tracking period may not cover seasonal or disaster-recovery usage patterns

## Data Sensitivity

- Unused permission findings: MEDIUM — reveals IAM policy details and usage patterns
- CloudTrail cross-reference: HIGH — reveals API call history and principal activity
- Policy generation results: MEDIUM — reveals minimum required permissions

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER remove permissions from production roles without testing in a non-production environment first
- NEVER assume a permission is truly unused without verifying the tracking period covers all usage patterns

## Phase 3 — Rollback

1. If permissions were removed incorrectly, restore the IAM policy: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://original-policy.json`
2. For managed policies, reattach: `aws iam attach-role-policy --role-name <role> --policy-arn <policy-arn>`
3. If permission boundaries were applied too restrictively, update: `aws iam put-role-permissions-boundary --role-name <role> --permissions-boundary <less-restrictive-policy-arn>`
4. Verify affected workloads are functioning correctly after rollback
5. Extend the tracking period before re-attempting permission reduction

## Output Format

```yaml
root_cause: "unused_permissions — <specific_cause>"
evidence:
  - type: permission_findings
    content: "<unused permissions and usage analysis>"
severity: MEDIUM
mitigation:
  immediate: "Review and right-size permissions"
  long_term: "Implement least-privilege with regular reviews"
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
