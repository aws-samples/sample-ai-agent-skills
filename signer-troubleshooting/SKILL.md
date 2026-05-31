---
name: signer-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Signer problems
  by analyzing signing profiles, signing jobs, signature validation,
  Lambda code signing, IoT code signing, signing platforms, certificate
  issues, IAM permissions, cross-account signing, and notification
  configuration using structured runbooks.
  Activate when: signing profile failures, signing job errors, signature
  validation problems, Lambda code signing issues, IoT signing errors,
  platform compatibility issues, certificate problems, IAM permission
  errors, cross-account signing failures, or the user says something
  is wrong with AWS Signer.
compatibility: >
  Requires AWS CLI or SDK access with signer, lambda, iot, iam,
  kms, sns, cloudtrail, and cloudwatch permissions.
---

# AWS Signer Diagnostics

## When to use

Any AWS Signer investigation — signing profile creation, profile revocation, signing job failures, signature validation, Lambda code signing, IoT code signing, signing platforms, certificate issues, IAM permissions, cross-account signing, or notification configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws signer list-signing-profiles --query 'profiles[*].{Name:profileName,Platform:platformId,Status:status,Version:profileVersion}'
aws signer list-signing-jobs --query 'jobs[*].{JobId:jobId,Status:status,ProfileName:profileName,CreatedAt:createdAt}'
aws lambda get-code-signing-config --code-signing-config-arn <config-arn>
```

### Step 2 — Domain deep dive

```
aws signer get-signing-profile --profile-name <profile-name>
aws signer describe-signing-job --job-id <job-id>
aws signer list-signing-platforms --query 'platforms[*].{PlatformId:platformId,DisplayName:displayName,Category:category}'
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=signer.amazonaws.com --max-results 20
aws lambda list-code-signing-configs --query 'CodeSigningConfigs[*].{Arn:CodeSigningConfigArn,AllowedPublishers:AllowedPublishers,Policies:CodeSigningPolicies}'
aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names signer:StartSigningJob signer:GetSigningProfile
```

Read `references/guardrails.md` before concluding on any AWS Signer issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `signer list-signing-profiles` | List signing profiles |
| `signer get-signing-profile` | Get profile details |
| `signer list-signing-jobs` | List signing jobs |
| `signer describe-signing-job` | Get job details |
| `signer list-signing-platforms` | List available platforms |
| `lambda get-code-signing-config` | Get Lambda signing config |
| `signer revoke-signing-profile` | Revoke a signing profile |

## Gotchas: AWS Signer

- Signing profiles are immutable once created. You cannot modify a profile's platform or signing parameters. Create a new profile with the desired settings instead.
- Revoking a signing profile invalidates ALL signatures made with that profile. This can break deployed Lambda functions and IoT firmware if they require valid signatures.
- Lambda code signing configs have two enforcement modes: Warn (log but allow) and Enforce (block unsigned/invalid code). Enforce mode can prevent deployments.
- Signing jobs are asynchronous. Large artifacts may take minutes to sign. Job status transitions from InProgress to Succeeded or Failed.
- AWS Signer uses AWS KMS for key management. The signing key is managed by AWS — you cannot bring your own signing key for most platforms.
- Cross-account signing requires the signing profile to be shared via the profile's resource policy. The signing account must grant signer:StartSigningJob to the target account.
- IoT code signing has different platforms than Lambda code signing. Ensure you use the correct platform ID for your use case (AmazonFreeRTOS, AWSIoTDeviceManagement).

## Anti-hallucination rules

1. Always cite specific profile names, job IDs, or API responses as evidence.
2. Signing profiles are immutable — never suggest modifying an existing profile's platform or parameters.
3. Profile revocation is irreversible and affects all existing signatures. Never suggest revoking without understanding the blast radius.
4. Lambda code signing and IoT code signing use different platforms. Never conflate them.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Profiles | A1–A2 | Signing profile creation, profile revocation |
| B — Jobs | B1–B2 | Signing job failures, signature validation errors |
| C — Lambda | C1–C2 | Lambda code signing, IoT code signing |
| D — Platform | D1–D2 | Signing platform issues, certificate issues |
| E — Access | E1–E2 | IAM permissions, cross-account signing |
| F — Notifications | F1 | Notification configuration |
| Z — Catch-All | Z1 | General troubleshooting |
