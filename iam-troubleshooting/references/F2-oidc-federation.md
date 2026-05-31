---
title: "F2 — OIDC Federation Issues"
description: "Diagnose OIDC provider configuration, audience, and thumbprint issues"
status: active
severity: HIGH
triggers:
  - "OIDC"
  - "OpenID Connect"
  - "OIDC provider"
  - "IRSA"
  - "GitHub Actions OIDC"
  - "thumbprint"
owner: devops-agent
objective: "Identify and fix OIDC federation configuration issues"
context: "OIDC federation requires: (1) OIDC provider registered in IAM with URL, audience, and thumbprint, (2) role trust policy allowing the OIDC provider with conditions, (3) correct token claims. Common uses: EKS IRSA, GitHub Actions, GitLab CI, and other CI/CD systems."
---

## Phase 1 — Triage

MUST:
- List OIDC providers: `aws iam list-open-id-connect-providers`
- Get provider details: `aws iam get-open-id-connect-provider --open-id-connect-provider-arn <arn>`
- Check the role trust policy: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Verify the trust policy has the correct OIDC principal and conditions

SHOULD:
- Check CloudTrail for AssumeRoleWithWebIdentity events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity`
- Verify the OIDC provider URL matches the token issuer exactly (including https://)
- Check the audience (client_id) list matches the token's aud claim
- Verify the thumbprint is current (OIDC provider certificate may have rotated)

MAY:
- Decode the JWT token to inspect claims (iss, aud, sub)
- Check the OIDC provider's .well-known/openid-configuration endpoint
- Verify network connectivity to the OIDC provider URL

## Phase 2 — Remediate

MUST:
- If thumbprint is wrong: `aws iam update-open-id-connect-provider-thumbprint --open-id-connect-provider-arn <arn> --thumbprint-list <new-thumbprint>`
- If audience is missing: `aws iam add-client-id-to-open-id-connect-provider --open-id-connect-provider-arn <arn> --client-id <audience>`
- Trust policy must include:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Federated": "<oidc-provider-arn>"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "<provider-url>:aud": "<audience>",
        "<provider-url>:sub": "<subject>"
      }
    }
  }
  ```

SHOULD:
- Use StringLike for sub conditions when matching patterns (e.g., GitHub repo patterns)
- Scope conditions tightly to prevent unauthorized token holders from assuming the role
- For EKS IRSA: ensure the EKS OIDC provider URL matches the cluster's OIDC issuer

MAY:
- Automate thumbprint rotation monitoring
- Use AWS IAM Roles Anywhere for on-premises workloads instead of OIDC

## Common Issues

- symptoms: "AssumeRoleWithWebIdentity fails with 'InvalidIdentityToken'"
  diagnosis: "The OIDC provider thumbprint doesn't match the current certificate, or the provider URL is wrong."
  resolution: "Update the thumbprint or verify the provider URL matches the token issuer exactly."

- symptoms: "GitHub Actions OIDC fails with 'Not authorized'"
  diagnosis: "The trust policy condition doesn't match the GitHub token's sub claim."
  resolution: "For GitHub: sub format is `repo:<org>/<repo>:ref:refs/heads/<branch>`. Use StringLike with wildcards for flexibility."

- symptoms: "EKS IRSA pod cannot assume role"
  diagnosis: "The EKS OIDC provider is not registered in IAM, or the service account annotation is wrong."
  resolution: "Verify the OIDC provider exists, the trust policy sub matches `system:serviceaccount:<namespace>:<sa-name>`, and the pod has the correct service account."

## Output Format

```yaml
root_cause: "oidc_federation — <specific_cause>"
evidence:
  - type: oidc_provider
    content: "<provider URL, audience, thumbprint>"
  - type: trust_policy
    content: "<role trust policy>"
  - type: token_claims
    content: "<relevant JWT claims>"
severity: HIGH
mitigation:
  immediate: "Fix OIDC provider config, thumbprint, or trust policy conditions"
  long_term: "Automate thumbprint rotation and implement condition monitoring"
```


## Safety Ratings

safety_ratings:
  - "OIDC federation configuration (provider thumbprint, audience, trust policy changes): RED — OIDC changes affect workload identity federation (EKS IRSA, GitHub Actions, CI/CD); misconfiguration can break all workloads using the OIDC provider"

## Escalation Conditions

escalation_conditions:
  - "ALWAYS escalate to a human before modifying OIDC provider configuration or trust policies"
  - "Escalate if the OIDC provider is used by EKS IRSA across multiple namespaces or clusters"
  - "Escalate if the change involves modifying trust policy conditions that gate CI/CD pipeline access"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original OIDC provider thumbprint using `aws iam update-open-id-connect-provider-thumbprint --open-id-connect-provider-arn <arn> --thumbprint-list <original-thumbprint>`, and restore the original trust policy using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-open-id-connect-provider` returns OIDC provider URL, audiences, and thumbprints — classification: HIGH
- `aws iam get-role` returns trust policy documents revealing OIDC federation conditions — classification: HIGH
- JWT token claims (if decoded during troubleshooting) may contain workload identity details — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during OIDC federation troubleshooting

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
