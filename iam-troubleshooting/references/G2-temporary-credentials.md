---
title: "G2 — Temporary Credentials (STS)"
description: "Diagnose issues with STS temporary credentials, expiry, and refresh"
status: active
severity: MEDIUM
triggers:
  - "temporary credentials"
  - "STS token"
  - "expired token"
  - "ExpiredTokenException"
  - "session token"
  - "security token expired"
owner: devops-agent
objective: "Resolve issues with STS temporary credential lifecycle and refresh"
context: "STS temporary credentials (access key, secret key, session token) have a limited lifetime. They are issued by AssumeRole, GetSessionToken, AssumeRoleWithSAML, AssumeRoleWithWebIdentity, and GetFederationToken. Expired credentials must be refreshed — they cannot be extended."
---

## Phase 1 — Triage

MUST:
- Check current credentials: `aws sts get-caller-identity`
- Verify if credentials are temporary (session token present) or long-term
- Check the error message for ExpiredTokenException or ExpiredToken
- Identify how the credentials were obtained (AssumeRole, instance profile, etc.)

SHOULD:
- Check the credential expiry time (available in the AssumeRole response or instance metadata)
- For EC2: check instance metadata for credential refresh: `curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>`
- Verify the application's credential provider chain is configured correctly

MAY:
- Check CloudTrail for the original credential issuance event
- Review SDK configuration for credential refresh settings

## Phase 2 — Remediate

MUST:
- If credentials are expired: obtain new credentials through the same mechanism
  - AssumeRole: `aws sts assume-role --role-arn <arn> --role-session-name <name>`
  - GetSessionToken (with MFA): `aws sts get-session-token --serial-number <mfa-arn> --token-code <code>`
- Ensure the application refreshes credentials before expiry

SHOULD:
- Use AWS SDK credential providers that handle automatic refresh (instance profiles, IRSA, ECS task roles)
- Set appropriate session duration to minimize refresh frequency
- Implement credential refresh logic with buffer time (refresh 5-10 minutes before expiry)

MAY:
- Use credential caching with automatic refresh in custom applications
- Monitor credential refresh failures with CloudWatch

## Common Issues

- symptoms: "ExpiredTokenException after running a long job"
  diagnosis: "The temporary credentials expired during the job. Default session is 1 hour."
  resolution: "Request longer session duration or implement credential refresh during the job."

- symptoms: "EC2 instance suddenly gets AccessDenied"
  diagnosis: "Instance profile credentials may have failed to refresh. Check instance metadata service."
  resolution: "Verify IMDSv2 is accessible. Check the instance role still exists and has permissions."

- symptoms: "Credentials work in one SDK call but fail in the next"
  diagnosis: "Credentials expired between calls. The SDK may not be refreshing automatically."
  resolution: "Use the SDK's built-in credential provider chain instead of manually setting credentials."

## Output Format

```yaml
root_cause: "temporary_credentials — <specific_cause>"
evidence:
  - type: credential_source
    content: "<AssumeRole|GetSessionToken|instance_profile|etc>"
  - type: expiry
    content: "<expiry timestamp>"
  - type: error
    content: "<ExpiredTokenException details>"
severity: MEDIUM
mitigation:
  immediate: "Refresh credentials through the original mechanism"
  long_term: "Use SDK credential providers with automatic refresh"
```


## Safety Ratings

safety_ratings:
  - "Temporary credentials analysis (read-only investigation of STS credential lifecycle): GREEN — All triage commands are read-only API calls; remediation involves refreshing credentials rather than modifying IAM resources"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the credential refresh mechanism is broken in a production workload"
  - "Escalate if the issue involves instance metadata service (IMDS) failures on production EC2 instances"
  - "Escalate if expired credentials are causing cascading failures across multiple services"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Revert any SDK or application configuration changes to the original credential provider settings
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws sts get-caller-identity` returns the caller's account, principal ARN, and session details — classification: MEDIUM
- Instance metadata credentials (`curl http://169.254.169.254/latest/meta-data/iam/security-credentials/`) return temporary access keys and session tokens — classification: HIGH
- `aws cloudtrail lookup-events` for AssumeRole events reveals credential issuance patterns — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- No temporary resources are created during temporary credentials analysis

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
