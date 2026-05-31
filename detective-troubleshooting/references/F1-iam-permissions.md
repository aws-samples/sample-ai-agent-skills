---
title: "F1 — IAM Permission Issues"
description: "Diagnose Detective IAM permission problems including service-linked roles and policy configuration"
status: active
severity: HIGH
triggers:
  - "detective permissions"
  - "access denied detective"
  - "iam detective"
  - "detective role"
  - "permission error"
owner: devops-agent
objective: "Resolve IAM permission issues for Detective operations"
context: "Detective requires specific IAM permissions for admin operations, member operations, and investigation. It also uses a service-linked role (AWSServiceRoleForDetective) for data access. Common issues include missing permissions and SCP restrictions."
---

## Phase 1 — Triage

MUST:
- Verify caller identity: `aws sts get-caller-identity`
- Check IAM permissions for Detective actions
- Verify service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForDetective`
- Check CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=detective.amazonaws.com --max-results 10`

SHOULD:
- Check for SCP restrictions on Detective operations
- Verify Organizations permissions if using delegated admin
- Check if the user/role has the correct Detective managed policy

MAY:
- Use IAM Policy Simulator to test permissions
- Check for permission boundaries limiting Detective access

## Phase 2 — Remediate

MUST:
- Add required Detective permissions to IAM policy (e.g., `AmazonDetectiveFullAccess` or custom policy)
- If service-linked role missing, it will be created automatically on first use
- If SCP blocking, update SCP to allow Detective operations

SHOULD:
- Use managed policies where possible for easier maintenance
- Separate admin and read-only permissions

## Output Format

```yaml
root_cause: "iam_permissions — <specific_cause>"
evidence:
  - type: caller_identity
    content: "<sts get-caller-identity>"
  - type: access_denied
    content: "<CloudTrail denied events>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or SCP restrictions"
  long_term: "Use managed policies and implement least-privilege access"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — IAM permissions control access to security investigation data |
| Blast Radius | MEDIUM — Permission changes affect who can investigate security incidents |
| Rollback Complexity | LOW — IAM policy changes can be reverted |
| Compliance Scope | CRITICAL — Access to security investigation tools must be controlled |

## Escalation Conditions

- Escalate to security team if access denied errors block active incident investigations
- Escalate to IAM administrators if SCP restrictions affect Detective operations
- Escalate to compliance if Detective access is not properly audited

## Data Sensitivity

- **IAM policies** for Detective control access to security data — security-critical
- **Service-linked role** has broad data access — do not modify
- **CloudTrail access denied events** reveal permission gaps

## PROHIBITED Actions

- Do NOT grant Detective full access to users who don't need security investigation capabilities
- Do NOT modify the AWSServiceRoleForDetective service-linked role
- Do NOT remove Detective permissions from security analysts during active investigations

## Phase 3 — Rollback

- Revert IAM policy changes to previous version
- Restore SCP settings if they were modified
- Verify security analysts can access Detective after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
