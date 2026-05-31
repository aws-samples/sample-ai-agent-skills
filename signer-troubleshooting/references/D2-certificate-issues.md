---
title: "D2 — Certificate Issues"
description: "Diagnose certificate-related issues in AWS Signer workflows"
status: active
severity: HIGH
triggers: ["certificate error", "signing certificate issue", "certificate validation failed"]
owner: devops-agent
objective: "Resolve certificate-related issues affecting signing operations"
context: "AWS Signer manages signing certificates internally. For most platforms, certificates are auto-generated and managed. Certificate issues typically relate to validity periods, trust chains, or cross-service integration."
---

## Phase 1 — Triage

MUST:
- Check signing profile details including certificate info: `aws signer get-signing-profile --profile-name <profile-name>`
- Verify the signing profile is not revoked or expired
- Check the signature validity period configuration

SHOULD:
- Verify the signing certificate chain is trusted by the validating service
- Check if the certificate has expired

MAY:
- Review ACM Private CA if using custom certificates for specific platforms
- Check CloudTrail for certificate-related errors

## Phase 2 — Remediate

MUST:
- If certificate expired: create a new signing profile (certificates are tied to profiles)
- If trust chain issue: verify the validating service trusts AWS Signer certificates
- Ensure the signing profile's validity period has not been exceeded

SHOULD:
- Monitor certificate expiration dates
- Plan profile rotation before certificate expiry

MAY:
- For custom certificate workflows, verify ACM Private CA configuration
- Set up CloudWatch alarms for certificate expiration

## Common Issues

- symptoms: "Signature validation fails with 'certificate not trusted'"
  diagnosis: "Validating service does not trust the AWS Signer certificate chain."
  resolution: "Ensure the service is configured to trust AWS Signer certificates. For Lambda, this is automatic."

- symptoms: "Signing profile shows expired status"
  diagnosis: "Profile's signature validity period has been exceeded."
  resolution: "Create a new signing profile. Re-sign artifacts with the new profile."

## Safety Ratings

```
safety_ratings:
  - "get-signing-profile: GREEN — read-only profile and certificate details"
  - "put-signing-profile: YELLOW — creates new profile with new certificate"
  - "acm-pca issue-certificate: YELLOW — issues new certificate from private CA"
  - "revoke-signing-profile: RED — permanently invalidates profile and certificate"
```

## Escalation Conditions

- Certificate expiration imminent and profile rotation not planned
- Custom certificate chain required but ACM Private CA not configured
- Certificate trust issues between AWS Signer and third-party validation
- Multiple profiles expiring simultaneously

## Data Sensitivity

- Signing profile certificate details: MEDIUM — reveals certificate metadata
- ACM Private CA configuration: HIGH — reveals PKI infrastructure
- Certificate chain: LOW — public certificate information
- Profile validity periods: LOW — configuration metadata

## Prohibited Actions

- NEVER extract or attempt to export AWS Signer managed certificates
- NEVER ignore certificate expiration warnings
- NEVER use self-signed certificates where AWS Signer certificates are required
- NEVER revoke a signing profile to fix certificate issues without creating a replacement first

## Phase 3 — Rollback

1. If a new profile was created with incorrect certificate settings, cancel it
2. If ACM Private CA was modified, revert configuration
3. Restore previous signing profile references in code signing configs
4. Verify signature validation works with the correct certificate chain
5. Document certificate lifecycle and rotation schedule

## Output Format

```yaml
root_cause: "certificate_issue — <specific_cause>"
evidence:
  - type: certificate_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix certificate issue"
  long_term: "Implement certificate lifecycle monitoring"
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
