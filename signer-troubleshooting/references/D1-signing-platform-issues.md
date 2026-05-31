---
title: "D1 — Signing Platform Issues"
description: "Diagnose signing platform compatibility and availability issues"
status: active
severity: MEDIUM
triggers: ["platform not found", "wrong platform", "platform compatibility error"]
owner: devops-agent
objective: "Resolve signing platform selection and compatibility issues"
context: "AWS Signer supports multiple platforms for different use cases. Each platform defines the signing algorithm, artifact format, and signature format. Using the wrong platform causes signing failures."
---

## Phase 1 — Triage

MUST:
- List available platforms: `aws signer list-signing-platforms --query 'platforms[*].{Id:platformId,DisplayName:displayName,Category:category,MaxSize:maxSizeInMB}'`
- Check current profile's platform: `aws signer get-signing-profile --profile-name <profile-name> --query '{Platform:platformId,Status:status}'`
- Verify the platform supports the artifact type

SHOULD:
- Check platform availability in the current region
- Verify the artifact size is within platform limits

MAY:
- Compare platform capabilities for the use case
- Check if a newer platform version is available

## Phase 2 — Remediate

MUST:
- Select the correct platform for the use case:
  - Lambda: `AWSLambda-SHA384-ECDSA`
  - IoT Device Management: `AWSIoTDeviceManagement-SHA256-ECDSA`
  - FreeRTOS: `AmazonFreeRTOS-TI-CC3220SF`
- Create a new profile with the correct platform if the current one is wrong

SHOULD:
- Document the platform selection rationale
- Verify the platform is available in all required regions

MAY:
- Test signing with the new platform before production use
- Set up platform compatibility checks in CI/CD

## Common Issues

- symptoms: "PutSigningProfile fails with 'Platform not found'"
  diagnosis: "Invalid platform ID or platform not available in the region."
  resolution: "Use list-signing-platforms to get valid IDs. Check regional availability."

- symptoms: "Signing succeeds but artifact is not compatible with the target service"
  diagnosis: "Wrong platform selected — signature format doesn't match service expectations."
  resolution: "Create a new profile with the correct platform for the target service."

## Safety Ratings

```
safety_ratings:
  - "list-signing-platforms: GREEN — read-only platform listing"
  - "get-signing-profile: GREEN — read-only profile details"
  - "put-signing-profile: YELLOW — creates new signing profile"
  - "cancel-signing-profile: RED — cancels profile permanently"
```

## Escalation Conditions

- Required platform not available in the target region
- Platform limitations prevent signing the artifact (size, format)
- Multiple platforms needed for different deployment targets
- Platform deprecation affecting existing signing workflows

## Data Sensitivity

- `aws signer list-signing-platforms`: LOW — public platform information
- Platform selection: LOW — technical configuration choice
- Profile creation: MEDIUM — creates cryptographic signing resource
- Artifact format requirements: LOW — technical specifications

## Prohibited Actions

- NEVER guess platform IDs — always use list-signing-platforms to verify
- NEVER use IoT platforms for Lambda signing or vice versa
- NEVER create profiles with deprecated platforms
- NEVER assume platform availability is the same across all regions

## Phase 3 — Rollback

1. If a profile was created with the wrong platform, cancel it
2. Create a new profile with the correct platform
3. Update all signing job configurations to use the new profile
4. Verify signing works correctly with the new platform
5. Document the platform selection for future reference

## Output Format

```yaml
root_cause: "signing_platform — <specific_cause>"
evidence:
  - type: platform_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Select correct signing platform"
  long_term: "Document platform requirements per use case"
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
