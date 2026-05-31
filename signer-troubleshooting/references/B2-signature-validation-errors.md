---
title: "B2 — Signature Validation Errors"
description: "Diagnose signature validation failures during deployment"
status: active
severity: HIGH
triggers: ["signature validation failed", "invalid signature", "code signing validation error"]
owner: devops-agent
objective: "Resolve signature validation errors during Lambda deployment or IoT updates"
context: "Signature validation occurs when deploying signed artifacts. Lambda validates signatures against code signing configs. Validation failures can be caused by expired signatures, revoked profiles, or mismatched signing configs."
---

## Phase 1 — Triage

MUST:
- Check the code signing config: `aws lambda get-code-signing-config --code-signing-config-arn <config-arn>`
- Verify the signing profile used is not revoked: `aws signer get-signing-profile --profile-name <profile-name>`
- Check the signing job that produced the artifact: `aws signer describe-signing-job --job-id <job-id>`

SHOULD:
- Verify the signing profile ARN is in the code signing config's allowed publishers
- Check if the signature has expired (past the validity period)

MAY:
- Review CloudTrail for UpdateFunctionCode events with validation errors
- Check if the artifact was modified after signing (hash mismatch)

## Phase 2 — Remediate

MUST:
- If profile was revoked: create new profile, re-sign artifact, update code signing config
- If signature expired: re-sign with a valid profile
- If profile not in allowed publishers: add profile ARN to code signing config

SHOULD:
- Update code signing config: `aws lambda update-code-signing-config --code-signing-config-arn <arn> --allowed-publishers SigningProfileVersionArns=<profile-version-arn>`
- Verify the enforcement mode (Warn vs Enforce) to understand impact

MAY:
- Temporarily switch to Warn mode while fixing: `aws lambda update-code-signing-config --code-signing-config-arn <arn> --code-signing-policies UntrustedArtifactOnDeployment=Warn`
- Set up monitoring for signature validation failures

## Common Issues

- symptoms: "Lambda deployment fails with 'Code signing configuration policy rejection'"
  diagnosis: "Artifact signed with a profile not in the code signing config's allowed publishers."
  resolution: "Add the signing profile version ARN to the code signing config's allowed publishers."

- symptoms: "Lambda deployment fails with 'Signature expired'"
  diagnosis: "Signature validity period has passed."
  resolution: "Re-sign the artifact with a valid signing profile and redeploy."

## Safety Ratings

```
safety_ratings:
  - "lambda get-code-signing-config: GREEN — read-only config details"
  - "signer get-signing-profile: GREEN — read-only profile details"
  - "lambda update-code-signing-config: YELLOW — modifies signing validation policy"
  - "update enforcement to Warn: RED — reduces security enforcement"
```

## Escalation Conditions

- Signature validation blocks critical production deployment
- Signing profile was revoked due to security incident and re-signing is complex
- Multiple Lambda functions affected by expired signatures simultaneously
- Code signing config changes require security team approval

## Data Sensitivity

- `aws lambda get-code-signing-config`: MEDIUM — reveals signing policy and allowed publishers
- `aws signer describe-signing-job`: MEDIUM — reveals artifact locations and signing details
- Code signing config changes: MEDIUM — modify security validation policy
- Enforcement mode changes: HIGH — affect deployment security controls

## Prohibited Actions

- NEVER permanently disable code signing enforcement to bypass validation errors
- NEVER deploy unsigned code to production as a workaround
- NEVER remove all allowed publishers from a code signing config
- NEVER ignore signature validation errors — they may indicate tampering

## Phase 3 — Rollback

1. If code signing config was modified, revert to previous allowed publishers
2. If enforcement was changed to Warn, restore Enforce mode after fixing
3. If a new profile was added incorrectly, remove it from allowed publishers
4. Verify Lambda deployments work correctly after changes
5. Document the validation error and resolution

## Output Format

```yaml
root_cause: "signature_validation — <specific_cause>"
evidence:
  - type: validation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix signature validation issue"
  long_term: "Implement signature expiry monitoring and automated re-signing"
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
