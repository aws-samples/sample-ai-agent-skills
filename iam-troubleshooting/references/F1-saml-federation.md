---
title: "F1 — SAML Federation Issues"
description: "Diagnose SAML IdP configuration, assertion mapping, and trust policy issues"
status: active
severity: HIGH
triggers:
  - "SAML"
  - "SAML federation"
  - "IdP"
  - "identity provider"
  - "SSO SAML"
  - "federation login failed"
owner: devops-agent
objective: "Identify and fix SAML federation configuration issues preventing authentication"
context: "SAML federation requires: (1) SAML IdP configured in IAM with metadata, (2) role trust policy allowing the SAML provider, (3) correct SAML assertion attributes mapping roles and session duration. Common failures include metadata mismatch, wrong role ARN in assertion, and expired certificates."
---

## Phase 1 — Triage

MUST:
- List SAML providers: `aws iam list-saml-providers`
- Get SAML provider metadata: `aws iam get-saml-provider --saml-provider-arn <arn>`
- Check the role trust policy for SAML principal: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`
- Verify the trust policy has: `"Principal": {"Federated": "arn:aws:iam::<account>:saml-provider/<name>"}`

SHOULD:
- Check CloudTrail for AssumeRoleWithSAML events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithSAML`
- Verify the SAML assertion contains the correct Role attribute (role ARN + provider ARN)
- Check if the SAML metadata certificate has expired
- Verify the SAML assertion's Audience matches `https://signin.aws.amazon.com/saml`

MAY:
- Use a SAML debugging tool to inspect the assertion
- Check IdP logs for assertion generation errors
- Verify the SessionDuration attribute in the SAML assertion

## Phase 2 — Remediate

MUST:
- If metadata is outdated: `aws iam update-saml-provider --saml-provider-arn <arn> --saml-metadata-document file://metadata.xml`
- If trust policy is wrong, update it to include:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<account>:saml-provider/<name>"},
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": {"StringEquals": {"SAML:aud": "https://signin.aws.amazon.com/saml"}}
  }
  ```
- Ensure the SAML assertion Role attribute format is: `<role-arn>,<provider-arn>`

SHOULD:
- Update IdP certificate before expiry to avoid authentication outages
- Test federation in a staging environment before production changes
- Configure SessionDuration in the SAML assertion for appropriate session length

MAY:
- Implement AWS SSO/Identity Center instead of direct SAML federation for simpler management
- Set up monitoring for SAML provider certificate expiry

## Common Issues

- symptoms: "AssumeRoleWithSAML fails with 'Not authorized to perform sts:AssumeRoleWithSAML'"
  diagnosis: "The role trust policy doesn't allow the SAML provider or the condition doesn't match."
  resolution: "Verify the trust policy Principal matches the SAML provider ARN and the SAML:aud condition is correct."

- symptoms: "SAML login fails with 'RoleNotFound' or 'InvalidIdentityToken'"
  diagnosis: "The Role attribute in the SAML assertion has the wrong format or ARN."
  resolution: "The Role attribute must be: `arn:aws:iam::<account>:role/<role>,arn:aws:iam::<account>:saml-provider/<name>`"

- symptoms: "SAML federation worked yesterday but fails today"
  diagnosis: "The IdP signing certificate may have expired or been rotated."
  resolution: "Download new metadata from the IdP and update: `aws iam update-saml-provider --saml-provider-arn <arn> --saml-metadata-document file://new-metadata.xml`"

## Output Format

```yaml
root_cause: "saml_federation — <specific_cause>"
evidence:
  - type: saml_provider
    content: "<provider ARN and metadata status>"
  - type: trust_policy
    content: "<role trust policy>"
  - type: saml_assertion
    content: "<relevant assertion attributes>"
severity: HIGH
mitigation:
  immediate: "Fix SAML configuration (metadata, trust policy, or assertion mapping)"
  long_term: "Implement certificate rotation monitoring and consider AWS SSO"
```


## Safety Ratings

safety_ratings:
  - "SAML federation configuration (metadata updates, trust policy changes): RED — Federation changes affect authentication for all SAML-federated users; misconfiguration can lock out entire user populations"

## Escalation Conditions

escalation_conditions:
  - "ALWAYS escalate to a human before modifying SAML provider metadata or trust policies"
  - "Escalate if the SAML provider is used by multiple roles or applications"
  - "Escalate if the IdP certificate is expiring and rotation is needed"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original SAML provider metadata using `aws iam update-saml-provider --saml-provider-arn <arn> --saml-metadata-document file://original-metadata.xml`, and restore the original trust policy using `aws iam update-assume-role-policy --role-name <role> --policy-document file://original-trust-policy.json`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-saml-provider` returns SAML metadata including IdP certificates and endpoints — classification: HIGH
- `aws iam get-role` returns trust policy documents revealing federation configuration — classification: HIGH
- `aws cloudtrail lookup-events` for AssumeRoleWithSAML events reveals federated user identities and session details — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during SAML federation troubleshooting

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
