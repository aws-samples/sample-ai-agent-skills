---
title: "A2 — Signing Profile Revocation"
description: "Diagnose signing profile revocation issues and impact assessment"
status: active
severity: CRITICAL
triggers: ["revoke profile", "revocation impact", "invalidate signatures"]
owner: devops-agent
objective: "Safely manage signing profile revocation and assess impact"
context: "Profile revocation is PERMANENT and IRREVERSIBLE. It invalidates ALL signatures made with the profile. This affects deployed Lambda functions and IoT firmware. Always assess blast radius before revoking."
---

## Phase 1 — Triage

MUST:
- Check profile status: `aws signer get-signing-profile --profile-name <profile-name>`
- List signing jobs using this profile: `aws signer list-signing-jobs --platform-id <platform-id> --status Succeeded --query 'jobs[?profileName==\`<profile-name>\`]'`
- Identify Lambda functions using signatures from this profile: `aws lambda list-code-signing-configs`

SHOULD:
- Assess the blast radius: how many deployments use signatures from this profile
- Verify a replacement profile exists before revoking

MAY:
- Check IoT jobs that used this profile for firmware signing
- Review the reason for revocation (compromise, rotation, cleanup)

## Phase 2 — Remediate

MUST:
- Create a replacement profile BEFORE revoking: `aws signer put-signing-profile --profile-name <new-name> --platform-id <platform-id>`
- Re-sign all critical artifacts with the new profile
- Update code signing configs to reference the new profile

SHOULD:
- Revoke only after all deployments are migrated: `aws signer revoke-signing-profile --profile-name <name> --profile-version <version> --reason <reason> --effective-time <time>`
- Notify all teams using the profile before revocation

MAY:
- Schedule revocation during a maintenance window
- Use revoke-signature for individual signatures instead of full profile revocation

## Common Issues

- symptoms: "Lambda deployment fails after profile revocation"
  diagnosis: "Code signing config references the revoked profile and enforcement is enabled."
  resolution: "Update code signing config to reference the new profile. Re-sign and redeploy."

- symptoms: "Need to revoke profile but cannot identify all affected deployments"
  diagnosis: "No inventory of signed artifacts and their deployment locations."
  resolution: "Query signing jobs for the profile. Check all Lambda code signing configs. Audit IoT deployments."

## Safety Ratings

```
safety_ratings:
  - "get-signing-profile: GREEN — read-only profile details"
  - "list-signing-jobs: GREEN — read-only job listing"
  - "revoke-signing-profile: RED — permanently invalidates all signatures"
  - "revoke-signature: RED — permanently invalidates a specific signature"
```

## Escalation Conditions

- Profile revocation is needed due to key compromise (urgent security incident)
- Revocation would affect production Lambda functions or IoT devices
- Cannot identify all deployments using the profile's signatures
- Revocation required by compliance but replacement profile not ready

## Data Sensitivity

- `aws signer get-signing-profile`: LOW — profile metadata and status
- `aws signer list-signing-jobs`: MEDIUM — reveals signed artifact locations (S3 paths)
- Revocation reason: MEDIUM — may indicate security incident
- Affected deployment inventory: MEDIUM — reveals application architecture

## Prohibited Actions

- NEVER revoke a signing profile without first creating a replacement
- NEVER revoke a profile without assessing the blast radius on deployed applications
- NEVER revoke profiles during business hours without a maintenance window
- NEVER assume revocation only affects future signatures — it invalidates ALL past signatures

## Phase 3 — Rollback

1. Profile revocation CANNOT be rolled back — it is permanent
2. If revocation was premature, create a new profile immediately
3. Re-sign all affected artifacts with the new profile
4. Update all code signing configs and deployment pipelines
5. Document the incident and update the signing profile inventory

## Output Format

```yaml
root_cause: "signing_profile_revocation — <specific_cause>"
evidence:
  - type: revocation_impact
    content: "<affected deployments and blast radius>"
severity: CRITICAL
mitigation:
  immediate: "Create replacement profile and re-sign artifacts"
  long_term: "Implement signing profile lifecycle management"
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
