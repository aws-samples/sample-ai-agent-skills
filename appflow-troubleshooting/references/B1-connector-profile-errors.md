---
title: "B1 — Connector Profile Errors"
description: "Diagnose Amazon AppFlow connector profile creation and authentication errors"
status: active
severity: HIGH
triggers: ["connector profile error", "authentication failed", "connector profile creation failed"]
owner: devops-agent
objective: "Resolve connector profile issues including OAuth, API keys, and credential errors"
context: "Connector profiles store authentication credentials for SaaS sources. They are region-specific and support OAuth, API key, and basic auth."
---

## Phase 1 — Triage

MUST:
- List connector profiles: `aws appflow list-connector-profiles`
- Describe the profile: `aws appflow describe-connector-profiles --connector-profile-names <name>`
- Verify the connector type is supported: `aws appflow list-connectors`

SHOULD:
- Check if OAuth token has expired (for OAuth-based connectors)
- Verify API endpoint URL is correct and reachable

MAY:
- Review CloudTrail for connector profile API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateConnectorProfile --max-results 10`

## Phase 2 — Remediate

MUST:
- Re-authenticate OAuth if token expired
- Fix API endpoint URL or credentials
- Ensure IAM role has permissions for secrets access if using Secrets Manager

SHOULD:
- Test connectivity after fixing credentials
- Update all flows using the affected connector profile

MAY:
- Rotate credentials and update the connector profile

## Safety Ratings

```
safety_ratings:
  - "list-connector-profiles: GREEN — read-only enumeration"
  - "describe-connector-profiles: GREEN — read-only profile details"
  - "CreateConnectorProfile: YELLOW — stores credentials, non-destructive"
  - "UpdateConnectorProfile: YELLOW — modifies stored credentials"
```

## Escalation Conditions

- OAuth re-authentication fails repeatedly
- SaaS provider API endpoint is unreachable from AWS
- Connector profile requires credentials that are managed by another team
- Multiple flows are affected by a single connector profile failure

## Data Sensitivity

- `aws appflow describe-connector-profiles`: HIGH — reveals connector configuration and credential references
- `aws appflow list-connector-profiles`: MEDIUM — reveals SaaS integration inventory
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER log or display OAuth tokens, API keys, or credentials in troubleshooting output
- NEVER delete a connector profile that is referenced by active flows
- NEVER share connector profile credentials across AWS accounts
- NEVER use personal SaaS credentials for production connector profiles

## Phase 3 — Rollback

1. If connector profile was updated with wrong credentials, revert using update-connector-profile
2. If a new profile was created for testing, delete it: `aws appflow delete-connector-profile --connector-profile-name <name>`
3. Verify all flows using the profile are still functional
4. Confirm OAuth tokens are valid by running a test flow execution
5. Document credential rotation schedule

## Common Issues

- symptoms: "ConnectorAuthenticationException when running flow"
  diagnosis: "OAuth refresh token has expired or been revoked."
  resolution: "Re-authenticate the connector profile through the AppFlow console or re-create with fresh OAuth tokens."

## Output Format

```yaml
root_cause: "connector_profile — <specific_cause>"
evidence:
  - type: auth_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix authentication"
  long_term: "Monitor token expiry"
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
