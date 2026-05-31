---
title: "F1 — Proton IAM Role Issues"
description: "Diagnose IAM role permission problems across Proton operations"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "IAM role"
  - "permission denied"
  - "not authorized"
  - "service role"
owner: devops-agent
objective: "Identify and resolve IAM role permission issues affecting Proton operations"
context: "Proton uses multiple IAM roles: Proton service role, pipeline service role, component role, and environment provisioning role. Each has different trust policies and permission requirements. Permission errors are common when roles are misconfigured."
---

## Phase 1 — Triage

MUST:
- Check Proton service role: `aws proton get-account-settings`
- Verify service role trust policy: `aws iam get-role --role-name <proton-service-role> --query 'Role.AssumeRolePolicyDocument'`
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check specific policy: `aws iam get-policy-version --policy-arn <policy-arn> --version-id <version>`
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names <action> --resource-arns <resource-arn>`

SHOULD:
- Check pipeline role if pipeline failures: look at service pipeline configuration
- Check component role if component failures
- Verify environment provisioning role for environment operations
- Check for permission boundaries or SCPs

MAY:
- Review CloudTrail for AccessDenied events
- Compare role permissions with Proton documentation requirements

## Phase 2 — Remediate

MUST:
- Ensure Proton service role trusts `proton.amazonaws.com`
- Grant CloudFormation permissions for AWS-managed provisioning
- Grant appropriate resource creation permissions for template resources

SHOULD:
- Use separate roles for each Proton function (service, pipeline, component)
- Follow least-privilege principle with resource-scoped permissions
- Document role requirements for each template

MAY:
- Use AWS managed policies as a starting point
- Implement role permission auditing

## Common Issues

- symptoms: "Environment provisioning fails with access denied"
  diagnosis: "Proton service role lacks CloudFormation or resource creation permissions."
  resolution: "Add required permissions to the Proton service role."

- symptoms: "Pipeline fails with permission error"
  diagnosis: "Pipeline role lacks CodeBuild, S3, or deployment permissions."
  resolution: "Add required permissions to the pipeline service role."

- symptoms: "Proton API calls fail with not authorized"
  diagnosis: "User/role calling Proton API lacks proton:* permissions."
  resolution: "Grant Proton API permissions to the calling identity."

## Output Format

```yaml
root_cause: "iam_role_issue — <specific_cause>"
evidence:
  - type: role_config
    content: "<role trust policy and permissions>"
  - type: permission_test
    content: "<simulate-principal-policy results>"
  - type: error_message
    content: "<specific access denied error>"
severity: HIGH
mitigation:
  immediate: "Add missing permissions to the appropriate role"
  long_term: "Implement role permission templates and regular audits"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — IAM roles control all Proton resource access
- Service Disruption Risk: MEDIUM — IAM issues prevent provisioning and management
- Compliance Impact: HIGH — IAM is fundamental to security compliance

## Escalation Conditions
- IAM role changes break Proton provisioning
- Overly permissive roles detected
- Cross-account role assumptions fail
- Service-linked role issues
- IAM audit reveals unauthorized access

## Data Sensitivity
- IAM policies — CONFIDENTIAL
- Role configurations — CONFIDENTIAL
- Trust policies — CONFIDENTIAL
- Access audit logs — CONFIDENTIAL
- Never expose IAM credentials or session tokens

## Prohibited Actions
- Do NOT attach AdministratorAccess to Proton roles
- Do NOT use wildcard resource ARNs
- Do NOT share IAM credentials
- Do NOT bypass IAM review processes
- Do NOT modify service-linked roles without understanding impact
- Do NOT ignore IAM audit findings

## Phase 3 — Rollback
- Restore previous IAM policy versions
- Revert trust policy changes
- Re-enable permission boundaries
- Revoke compromised credentials
- Verify Proton operations after IAM changes
- Document IAM changes and conduct security review

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
