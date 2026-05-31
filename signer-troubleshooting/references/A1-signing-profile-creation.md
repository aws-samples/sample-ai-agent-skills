---
title: "A1 — Signing Profile Creation"
description: "Diagnose signing profile creation failures"
status: active
severity: HIGH
triggers: ["cannot create profile", "signing profile failed", "profile creation error"]
owner: devops-agent
objective: "Resolve signing profile creation issues"
context: "Signing profiles define the platform, signing parameters, and validity period. Profiles are immutable once created. Each profile is tied to a specific signing platform."
---

## Phase 1 — Triage

MUST:
- List existing profiles: `aws signer list-signing-profiles --query 'profiles[*].{Name:profileName,Platform:platformId,Status:status}'`
- List available platforms: `aws signer list-signing-platforms --query 'platforms[*].{Id:platformId,DisplayName:displayName,Category:category}'`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names signer:PutSigningProfile`

SHOULD:
- Check if a profile with the same name already exists
- Verify the platform ID is valid for the intended use case

MAY:
- Review CloudTrail for PutSigningProfile errors
- Check service quotas for signing profiles

## Phase 2 — Remediate

MUST:
- Use a valid platform ID: `aws signer put-signing-profile --profile-name <name> --platform-id <platform-id>`
- Ensure IAM permissions include `signer:PutSigningProfile`
- Specify signature validity period if required

SHOULD:
- Use descriptive profile names that indicate purpose and platform
- Set appropriate signature validity periods (not too short, not too long)

MAY:
- Add tags for cost allocation and management
- Configure signing profile overrides if supported by the platform

## Common Issues

- symptoms: "PutSigningProfile returns ValidationException"
  diagnosis: "Invalid platform ID or profile name format."
  resolution: "Use list-signing-platforms to get valid platform IDs. Profile names must match [a-zA-Z0-9_]."

- symptoms: "Profile creation succeeds but signing jobs fail"
  diagnosis: "Wrong platform selected for the artifact type."
  resolution: "Verify platform matches use case: AWSLambda-SHA384-ECDSA for Lambda, IoT platforms for firmware."

## Safety Ratings

```
safety_ratings:
  - "list-signing-profiles: GREEN — read-only profile listing"
  - "list-signing-platforms: GREEN — read-only platform listing"
  - "put-signing-profile: YELLOW — creates signing profile with managed key"
  - "cancel-signing-profile: RED — cancels profile, preventing future signing"
```

## Escalation Conditions

- Profile creation fails with internal service errors
- Required platform is not available in the region
- Service quota for signing profiles reached
- Custom signing parameters needed that are not supported by available platforms

## Data Sensitivity

- `aws signer list-signing-profiles`: LOW — profile metadata only
- `aws signer list-signing-platforms`: LOW — public platform information
- `aws signer put-signing-profile`: MEDIUM — creates cryptographic signing resource
- Profile ARN: LOW — resource identifier

## Prohibited Actions

- NEVER create signing profiles with overly long validity periods without security review
- NEVER reuse profile names from revoked profiles (may cause confusion)
- NEVER create profiles in regions where they won't be used (unnecessary key creation)
- NEVER share signing profile credentials or ARNs with unauthorized parties

## Phase 3 — Rollback

1. If a profile was created with wrong settings, cancel it: `aws signer cancel-signing-profile --profile-name <name>`
2. Create a new profile with correct settings
3. Update all references (code signing configs, CI/CD pipelines) to the new profile
4. Verify signing jobs work with the new profile
5. Document the profile change and notify the development team

## Output Format

```yaml
root_cause: "signing_profile_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix profile creation issue"
  long_term: "Standardize profile creation via IaC"
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
