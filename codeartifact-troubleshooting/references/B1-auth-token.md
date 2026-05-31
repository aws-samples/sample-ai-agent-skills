---
title: "B1 — Auth Token Issues"
description: "Diagnose CodeArtifact authentication token problems including expiry, permissions, and tool configuration"
status: active
severity: HIGH
triggers:
  - "auth token expired"
  - "codeartifact login failed"
  - "401 unauthorized"
  - "authentication failed"
  - "token error"
owner: devops-agent
objective: "Resolve authentication token issues and establish reliable auth workflow"
context: "CodeArtifact uses temporary auth tokens (max 12 hours). Tokens are used by npm, pip, Maven, and other package managers. Common issues include token expiry, wrong domain/owner, and IAM permission failures."
---

## Phase 1 — Triage

MUST:
- Verify AWS credentials: `aws sts get-caller-identity`
- Test token generation: `aws codeartifact get-authorization-token --domain <domain> --query authorizationToken`
- Verify IAM permissions include `codeartifact:GetAuthorizationToken` and `sts:GetServiceBearerToken`
- Check if domain owner is specified correctly (needed for cross-account)

SHOULD:
- Verify the token is being used with the correct repository endpoint
- Check if the package manager is configured with the current token
- Verify `--domain-owner` parameter if domain is in another account

MAY:
- Check CloudTrail for GetAuthorizationToken failures
- Verify no SCP blocking `sts:GetServiceBearerToken`

## Phase 2 — Remediate

MUST:
- Refresh token: `aws codeartifact login --tool <npm|pip|twine> --domain <domain> --repository <repo>`
- If cross-account, include `--domain-owner <account-id>`
- Ensure IAM policy includes both `codeartifact:GetAuthorizationToken` and `sts:GetServiceBearerToken`

SHOULD:
- Add token refresh step at the start of CI/CD jobs
- Use `aws codeartifact login` instead of manual token configuration

## Common Issues

- symptoms: "401 Unauthorized on npm install"
  diagnosis: "Auth token expired or npm not configured with CodeArtifact registry."
  resolution: "Run aws codeartifact login --tool npm to refresh."

- symptoms: "GetAuthorizationToken access denied"
  diagnosis: "Missing sts:GetServiceBearerToken permission."
  resolution: "Add sts:GetServiceBearerToken to IAM policy."

## Output Format

```yaml
root_cause: "auth_token_issue — <specific_cause>"
evidence:
  - type: caller_identity
    content: "<sts get-caller-identity>"
  - type: token_test
    content: "<token generation result>"
severity: HIGH
mitigation:
  immediate: "Refresh auth token"
  long_term: "Automate token refresh in CI/CD pipelines"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Refresh token with aws codeartifact login | GREEN |
| Include --domain-owner for cross-account | GREEN |
| Ensure IAM policy includes required permissions | YELLOW |
| Add token refresh step in CI/CD jobs | GREEN |
| Use aws codeartifact login instead of manual config | GREEN |

## Escalation Conditions

- Fix requires modifying production IAM policies
- Fix involves changing CI/CD pipeline authentication configurations

## Data Sensitivity

- **HIGH**: auth tokens provide access to all packages in the repository
- Token generation requires IAM credentials that may have broad CodeArtifact access

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If IAM permissions were modified, revert to previous policy
2. If CI/CD token refresh was added, verify it does not interfere with existing auth
3. If package manager configuration was changed, restore previous .npmrc, pip.conf, or settings.xml
4. Verify rollback by confirming package operations work with the restored configuration

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
