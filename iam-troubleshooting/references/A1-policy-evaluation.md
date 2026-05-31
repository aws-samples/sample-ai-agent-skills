---
title: "A1 — IAM Policy Evaluation Logic"
description: "Trace the full IAM policy evaluation chain to determine why access is allowed or denied"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "why is this denied"
  - "policy evaluation"
  - "permission not working"
owner: devops-agent
objective: "Systematically trace through all policy evaluation layers to identify the exact cause of an access decision"
context: "IAM evaluates policies in a specific order: explicit deny → SCP → resource policy → permission boundary → session policy → identity policy. An explicit deny in ANY layer is final. Understanding this chain is critical for diagnosing access issues."
---

## Phase 1 — Triage

MUST:
- Identify the principal, action, and resource from the error or CloudTrail event
- Get caller identity: `aws sts get-caller-identity`
- Simulate the policy evaluation: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <action> --resource-arns <resource-arn>`
- List all attached policies for the principal:
  - User: `aws iam list-attached-user-policies --user-name <user>` and `aws iam list-user-policies --user-name <user>`
  - Role: `aws iam list-attached-role-policies --role-name <role>` and `aws iam list-role-policies --role-name <role>`
- Check group policies: `aws iam list-groups-for-user --user-name <user>` then `aws iam list-attached-group-policies --group-name <group>`

SHOULD:
- Get the full policy document for each attached policy: `aws iam get-policy-version --policy-arn <policy-arn> --version-id <version-id>`
- Get inline policy documents: `aws iam get-user-policy --user-name <user> --policy-name <name>` or `aws iam get-role-policy --role-name <role> --policy-name <name>`
- Check for permission boundaries: `aws iam get-user --user-name <user>` (look for PermissionsBoundary field)
- Check CloudTrail for the denied event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<api-call>`

MAY:
- Check SCPs if the account is in an Organization: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- Use IAM Access Analyzer for policy validation: `aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY`

## Phase 2 — Remediate

MUST:
- Walk the evaluation chain in order:
  1. Check for explicit Deny in all identity, resource, SCP, boundary, and session policies
  2. Check SCP allows the action (if in Organization)
  3. Check resource-based policy (can independently grant same-account access)
  4. Check permission boundary allows the action (if set)
  5. Check session policy allows the action (if AssumeRole used session policy)
  6. Check identity-based policy grants the action
- Fix the specific layer that is blocking access

SHOULD:
- Re-run simulate-principal-policy after the fix to confirm
- Apply least-privilege — only add the specific actions and resources needed
- Document which layer was the root cause

MAY:
- Enable CloudTrail data events for ongoing monitoring
- Set up IAM Access Analyzer to detect overly permissive policies

## Common Issues

- symptoms: "simulate-principal-policy returns allowed but actual call is denied"
  diagnosis: "simulate-principal-policy does not evaluate resource policies, SCPs, or VPC endpoint policies. The deny is in a layer the simulator doesn't cover."
  resolution: "Manually check resource policies, SCPs, and VPC endpoint policies. Use simulate-custom-policy with all policy documents for a complete simulation."

- symptoms: "Policy allows s3:* but specific S3 call is denied"
  diagnosis: "A permission boundary, SCP, or session policy is restricting the effective permissions."
  resolution: "Check all layers. Effective permissions = intersection of all applicable policy types."

- symptoms: "Added permission but it's still denied after a few seconds"
  diagnosis: "IAM changes are eventually consistent. Propagation can take several seconds."
  resolution: "Wait 30-60 seconds and retry. If still denied, the issue is in another policy layer."

## Output Format

```yaml
root_cause: "policy_evaluation — <layer>: <specific_cause>"
evidence:
  - type: simulation_result
    content: "<simulate-principal-policy output>"
  - type: policy_document
    content: "<relevant policy statement>"
  - type: cloudtrail_event
    content: "<denied event details>"
severity: HIGH
evaluation_chain:
  explicit_deny: "none found | <policy_arn>"
  scp: "allows | denies | not_applicable"
  resource_policy: "allows | denies | not_applicable"
  permission_boundary: "allows | denies | not_set"
  session_policy: "allows | denies | not_set"
  identity_policy: "allows | denies"
mitigation:
  immediate: "Fix the blocking policy layer"
  long_term: "Implement IAM Access Analyzer and policy simulation in CI/CD"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws iam simulate-principal-policy` | GREEN | Read-only diagnostic |
| `aws iam list-attached-user-policies` / `list-attached-role-policies` | GREEN | Read-only diagnostic |
| `aws iam list-user-policies` / `list-role-policies` | GREEN | Read-only diagnostic |
| `aws iam get-policy-version` | GREEN | Read-only diagnostic |
| `aws iam get-user-policy` / `get-role-policy` | GREEN | Read-only diagnostic |
| `aws iam get-user` / `aws iam get-role` | GREEN | Read-only diagnostic |
| `aws cloudtrail lookup-events` | GREEN | Read-only diagnostic |
| `aws organizations list-policies-for-target` | GREEN | Read-only diagnostic |
| `aws accessanalyzer validate-policy` | GREEN | Read-only diagnostic |
| Fix identity policy (attach/put policy) | YELLOW | Modifies effective permissions for the principal |
| Modify SCP or permission boundary | RED | Affects all principals under the OU or boundary scope |

## Escalation Conditions

- Remediation requires modifying IAM policies in a production account
- Fix involves granting cross-account access
- Root cause involves the management account
- Multiple principals are affected
- The denied action involves security-sensitive services (KMS, STS, Organizations)

## Rollback

1. **Before any change**: Document current state with `aws iam get-policy-version` or `aws iam get-role`
2. **After change**: Verify the fix resolves the issue without granting excessive permissions
3. **If change causes issues**: Revert to the previous policy version or detach the modified policy
4. **Cleanup**: Remove any temporary policies or test principals created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-account-authorization-details` | HIGH | Contains all IAM entities — redact from stored output |
| `get-policy-version` | HIGH | Contains full policy document — summarize, don't reproduce |
| `simulate-principal-policy` | MEDIUM | Shows effective permissions — summarize results |
| `list-*` | LOW | Resource listings — safe to include |

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest disabling MFA requirements as a troubleshooting step
- ALWAYS identify the specific denied action from CloudTrail before suggesting a fix
- ALWAYS use least-privilege: grant only the specific action on the specific resource

## Cleanup

- No temporary resources are created during policy evaluation analysis

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
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
