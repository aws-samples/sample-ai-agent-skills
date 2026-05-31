---
title: "B2 — Salesforce Connector"
description: "Diagnose Salesforce-specific connector issues in Amazon AppFlow"
status: active
severity: HIGH
triggers: ["Salesforce connector error", "Salesforce OAuth failed", "Salesforce API limit"]
owner: devops-agent
objective: "Resolve Salesforce-specific connector issues including OAuth, API limits, and object access"
context: "Salesforce connector uses OAuth 2.0 with Connected App. Subject to Salesforce API call limits and object-level security."
---

## Phase 1 — Triage

MUST:
- Check connector profile status: `aws appflow describe-connector-profiles --connector-profile-names <salesforce-profile>`
- Review recent execution errors: `aws appflow describe-flow-execution-records --flow-name <flow-name> --max-results 5`
- Verify Salesforce Connected App configuration (OAuth scopes, callback URL)

SHOULD:
- Check Salesforce API usage limits in Salesforce Setup > System Overview
- Verify Salesforce user has access to the required objects and fields

MAY:
- Test Salesforce API connectivity independently using curl or Postman
- Review Salesforce login history for the integration user

## Phase 2 — Remediate

MUST:
- Re-authenticate OAuth if refresh token expired
- Ensure Connected App has required scopes: `api`, `refresh_token`, `offline_access`
- Verify Salesforce user profile has object-level and field-level security permissions

SHOULD:
- Use a dedicated Salesforce integration user (not a personal account)
- Monitor Salesforce API usage to avoid hitting limits

MAY:
- Configure Salesforce Platform Events for event-driven flows
- Implement incremental data transfer using LastModifiedDate filters

## Safety Ratings

```
safety_ratings:
  - "describe-connector-profiles: GREEN — read-only profile details"
  - "describe-flow-execution-records: GREEN — read-only execution history"
  - "UpdateConnectorProfile (re-auth): YELLOW — modifies stored OAuth tokens"
  - "start-flow: YELLOW — triggers data transfer from Salesforce"
```

## Escalation Conditions

- Salesforce Connected App was deleted or modified by Salesforce admin
- Salesforce API limits are consistently exceeded and cannot be increased
- Salesforce org migration or sandbox refresh invalidated credentials
- Field-level security changes in Salesforce break existing flows

## Data Sensitivity

- `aws appflow describe-connector-profiles`: HIGH — reveals Salesforce org connection details
- `aws appflow describe-flow-execution-records`: MEDIUM — reveals data transfer volumes and errors
- Salesforce API responses: HIGH — may contain business-sensitive CRM data

## Prohibited Actions

- NEVER log Salesforce OAuth tokens or refresh tokens
- NEVER use Salesforce admin credentials for AppFlow integration
- NEVER bypass Salesforce field-level security by changing user profiles without approval
- NEVER increase Salesforce API limits without understanding cost implications

## Phase 3 — Rollback

1. If OAuth was re-authenticated with wrong credentials, re-authenticate with correct Salesforce user
2. If Salesforce Connected App settings were changed, revert OAuth scopes and callback URL
3. Verify flows using the Salesforce connector still execute successfully
4. Confirm no unauthorized data was transferred during troubleshooting
5. Document Salesforce configuration changes and notify Salesforce admin

## Common Issues

- symptoms: "INVALID_SESSION_ID error in flow execution"
  diagnosis: "Salesforce OAuth refresh token has expired or been revoked."
  resolution: "Re-authenticate the connector profile. Ensure Connected App has 'Refresh Token' scope and token policy allows indefinite refresh."

## Output Format

```yaml
root_cause: "salesforce_connector — <specific_cause>"
evidence:
  - type: salesforce_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Salesforce authentication"
  long_term: "Monitor OAuth token health"
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
