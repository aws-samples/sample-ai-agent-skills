---
title: "E1 — Cognito Integration"
description: "Diagnose Cognito Identity pool integration issues with RUM"
status: active
severity: HIGH
triggers: ["Cognito error", "identity pool issue", "authentication failed", "unauthenticated access denied"]
owner: devops-agent
objective: "Resolve Cognito Identity pool integration issues for RUM data collection"
context: "RUM uses Cognito Identity pools for browser client authentication. The pool must allow unauthenticated identities with rum:PutRumEvents permission."
---

## Phase 1 — Triage

MUST:
- Get app monitor Cognito configuration: `aws rum get-app-monitor --name <name> --query 'AppMonitor.AppMonitorConfiguration.IdentityPoolId'`
- Verify Identity pool exists and allows unauthenticated: `aws cognito-identity describe-identity-pool --identity-pool-id <pool-id>`
- Check the unauthenticated IAM role: `aws cognito-identity get-identity-pool-roles --identity-pool-id <pool-id>`

SHOULD:
- Verify the IAM role has rum:PutRumEvents permission
- Check if the Identity pool is in the same region as the app monitor
- Verify no SCP or permission boundary blocks the Cognito or RUM actions

MAY:
- Check CloudTrail for GetCredentialsForIdentity failures
- Verify the Identity pool is not at its identity limit
- Check if the IAM role trust policy allows cognito-identity.amazonaws.com

## Phase 2 — Remediate

MUST:
- Enable unauthenticated identities on the pool: `aws cognito-identity update-identity-pool --identity-pool-id <pool-id> --allow-unauthenticated-identities`
- Attach rum:PutRumEvents permission to the unauthenticated IAM role
- Ensure the IAM role trust policy allows cognito-identity.amazonaws.com

SHOULD:
- Scope the rum:PutRumEvents permission to the specific app monitor ARN
- Verify the Identity pool region matches the app monitor region

MAY:
- Set up monitoring for Cognito authentication failures
- Configure the Identity pool via IaC for consistency

## Common Issues

- symptoms: "Browser console shows 'NotAuthorizedException' from Cognito"
  diagnosis: "Identity pool does not allow unauthenticated identities or the IAM role lacks rum:PutRumEvents."
  resolution: "Enable unauthenticated access: `aws cognito-identity update-identity-pool --identity-pool-id <pool-id> --identity-pool-name <name> --allow-unauthenticated-identities`. Add rum:PutRumEvents to the unauthenticated role."

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "describe-identity-pool: GREEN — read-only Cognito details"
  - "update-identity-pool: RED — modifies authentication configuration"
  - "IAM role policy update: YELLOW — changes permissions"
```

## Escalation Conditions

- Cognito Identity pool is shared with other applications and changes may affect them
- IAM role changes require security team approval
- Identity pool at capacity and cannot create new identities
- Cross-account Cognito configuration required

## Data Sensitivity

- `aws cognito-identity describe-identity-pool`: MEDIUM — reveals authentication configuration
- `aws cognito-identity get-identity-pool-roles`: MEDIUM — reveals IAM role mappings
- IAM role policies: MEDIUM — reveals permission scope

## Prohibited Actions

- NEVER grant overly broad permissions (rum:*) to the unauthenticated role
- NEVER modify a shared Cognito Identity pool without checking impact on other applications
- NEVER disable authenticated identities to fix unauthenticated access issues
- NEVER expose Cognito Identity pool credentials in client-side code beyond the snippet configuration

## Phase 3 — Rollback

1. If Identity pool was modified, revert to previous configuration
2. If IAM role permissions were changed, restore original policy
3. If the Identity pool was created in error, delete it after removing from the app monitor
4. Verify RUM data collection resumes after Cognito fixes
5. Document all Cognito and IAM changes

## Output Format

```yaml
root_cause: "cognito_integration — <specific_cause>"
evidence:
  - type: cognito_error
    content: "<Cognito configuration and error details>"
severity: HIGH
mitigation:
  immediate: "Fix Cognito Identity pool configuration"
  long_term: "Automate Cognito setup via IaC with least-privilege IAM roles"
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
