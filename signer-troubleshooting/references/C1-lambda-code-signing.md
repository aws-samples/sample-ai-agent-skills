---
title: "C1 — Lambda Code Signing"
description: "Diagnose Lambda code signing configuration and deployment issues"
status: active
severity: HIGH
triggers: ["Lambda code signing failed", "code signing config error", "unsigned Lambda deployment"]
owner: devops-agent
objective: "Resolve Lambda code signing configuration and deployment issues"
context: "Lambda code signing uses code signing configs to validate deployment packages. Configs specify allowed signing profiles and enforcement mode (Warn or Enforce). The platform is AWSLambda-SHA384-ECDSA."
---

## Phase 1 — Triage

MUST:
- List code signing configs: `aws lambda list-code-signing-configs --query 'CodeSigningConfigs[*].{Arn:CodeSigningConfigArn,AllowedPublishers:AllowedPublishers,Policy:CodeSigningPolicies}'`
- Check function's code signing config: `aws lambda get-function --function-name <name> --query 'Configuration.CodeSigningConfigArn'`
- Verify signing profile is Lambda-compatible: `aws signer get-signing-profile --profile-name <name> --query '{Platform:platformId,Status:status}'`

SHOULD:
- Check enforcement mode (Warn vs Enforce)
- Verify the signing profile version ARN matches the allowed publishers list

MAY:
- Review recent deployment attempts in CloudTrail
- Check if the Lambda function was recently updated to add code signing

## Phase 2 — Remediate

MUST:
- Create code signing config if missing: `aws lambda create-code-signing-config --allowed-publishers SigningProfileVersionArns=<profile-version-arn> --code-signing-policies UntrustedArtifactOnDeployment=Enforce`
- Attach config to function: `aws lambda put-function-code-signing-config --function-name <name> --code-signing-config-arn <config-arn>`
- Sign deployment package before deploying: `aws signer start-signing-job --source 's3={bucketName=<bucket>,key=<key>,version=<version>}' --destination 's3={bucketName=<bucket>,prefix=signed/}' --profile-name <profile-name>`

SHOULD:
- Use Enforce mode for production functions
- Include all required signing profile version ARNs in allowed publishers

MAY:
- Integrate signing into CI/CD pipeline
- Set up CloudWatch alarms for code signing validation failures

## Common Issues

- symptoms: "Lambda deployment rejected with 'Code signing configuration policy rejection'"
  diagnosis: "Deployment package not signed or signed with a profile not in allowed publishers."
  resolution: "Sign the package with an allowed profile. Or add the profile to the code signing config."

- symptoms: "Code signing config attached but unsigned code deploys successfully"
  diagnosis: "Enforcement mode is set to Warn, not Enforce."
  resolution: "Update to Enforce mode: aws lambda update-code-signing-config --code-signing-policies UntrustedArtifactOnDeployment=Enforce."

## Safety Ratings

```
safety_ratings:
  - "lambda list-code-signing-configs: GREEN — read-only listing"
  - "lambda get-function: GREEN — read-only function details"
  - "lambda create-code-signing-config: YELLOW — creates signing validation policy"
  - "lambda put-function-code-signing-config: YELLOW — enforces signing on function"
```

## Escalation Conditions

- Code signing enforcement blocks critical production deployment
- CI/CD pipeline not configured for signing and enforcement is enabled
- Multiple functions need code signing configs applied simultaneously
- Signing profile used by the pipeline was revoked or expired

## Data Sensitivity

- `aws lambda list-code-signing-configs`: MEDIUM — reveals signing policy
- `aws lambda get-function`: MEDIUM — reveals function configuration
- Code signing config: MEDIUM — reveals allowed signing profiles
- Deployment packages: HIGH — contain application code

## Prohibited Actions

- NEVER disable code signing enforcement permanently to work around deployment issues
- NEVER deploy unsigned code to production Lambda functions
- NEVER remove all allowed publishers from a code signing config
- NEVER use Warn mode in production without a plan to migrate to Enforce

## Phase 3 — Rollback

1. If code signing config was attached incorrectly, remove it: `aws lambda delete-function-code-signing-config --function-name <name>`
2. If enforcement mode was changed, revert: `aws lambda update-code-signing-config --code-signing-config-arn <arn> --code-signing-policies UntrustedArtifactOnDeployment=<original-mode>`
3. If allowed publishers were modified, restore original list
4. Verify Lambda deployments work correctly after changes
5. Document the code signing configuration changes

## Output Format

```yaml
root_cause: "lambda_code_signing — <specific_cause>"
evidence:
  - type: code_signing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Lambda code signing configuration"
  long_term: "Integrate signing into CI/CD pipeline"
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
