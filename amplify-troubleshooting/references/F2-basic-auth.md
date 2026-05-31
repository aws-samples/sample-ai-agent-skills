---
title: "F2 — Basic Auth Issues"
description: "Diagnose basic authentication configuration problems"
status: active
severity: LOW
triggers:
  - "basic auth not working"
  - "password protection"
  - "basic auth prompt"
  - "staging protection"
  - "branch password"
owner: devops-agent
objective: "Identify and resolve basic authentication issues"
context: "Basic auth issues include: auth not enabled, credentials incorrect, auth applying to wrong branch, or auth not protecting as expected. Basic auth protects the entire branch — it cannot be scoped to specific paths."
---

## Phase 1 — Triage

MUST:
- Check branch basic auth settings: `aws amplify get-branch --app-id <app-id> --branch-name <branch> --query 'branch.{BasicAuthEnabled:enableBasicAuth,Credentials:basicAuthCredentials}'`
- Verify basic auth is enabled on the correct branch
- Test with correct credentials

SHOULD:
- Check if credentials are base64 encoded correctly
- Verify basic auth is not conflicting with application auth
- Check if CDN cache is serving unauthenticated content

MAY:
- Test with curl: `curl -u username:password https://<branch>.<app-id>.amplifyapp.com`
- Check if custom domain has different auth settings

## Phase 2 — Remediate

MUST:
- Enable basic auth: `aws amplify update-branch --app-id <app-id> --branch-name <branch> --enable-basic-auth --basic-auth-credentials $(echo -n 'user:pass' | base64)`
- Set correct credentials (base64 encoded user:password)

SHOULD:
- Enable on staging/preview branches, disable on production
- Clear CDN cache after enabling auth

MAY:
- Use different credentials per branch
- Implement application-level auth for path-specific protection

## Safety Ratings

```
safety_ratings:
  - "get-branch (basicAuth settings): GREEN — read-only auth check"
  - "test with curl: GREEN — read-only connectivity test"
  - "update-branch (enable basic auth): YELLOW — adds authentication requirement"
  - "update-branch (disable basic auth): YELLOW — removes authentication protection"
```

## Escalation Conditions

- Basic auth accidentally disabled on a staging branch with sensitive data
- Basic auth credentials compromised and need rotation
- Basic auth conflicting with application-level authentication
- CDN cache serving unauthenticated content after enabling auth

## Data Sensitivity

- Basic auth credentials: HIGH — username and password (base64 encoded)
- Branch auth configuration: LOW — enabled/disabled status
- CDN cache status: LOW — operational metadata

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER disable basic auth on staging branches without alternative protection
- NEVER use weak or default credentials for basic auth

## Phase 3 — Rollback

1. If basic auth was disabled incorrectly, re-enable it: `aws amplify update-branch --app-id <id> --branch-name <branch> --enable-basic-auth --basic-auth-credentials <base64-creds>`
2. If credentials were changed incorrectly, restore previous credentials
3. Clear CDN cache after any auth changes: trigger a redeployment
4. Verify basic auth is working by testing with curl
5. Document the auth configuration for the team

## Output Format

```yaml
root_cause: "basic_auth_issue — <specific_cause>"
evidence:
  - type: auth_config
    content: "<basic auth settings>"
  - type: branch
    content: "<branch name and stage>"
severity: LOW
mitigation:
  immediate: "Enable basic auth with correct credentials"
  long_term: "Implement proper access control strategy per environment"
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
