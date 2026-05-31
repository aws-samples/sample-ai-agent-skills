---
title: "A2 — Portal Configuration"
description: "Diagnose WorkSpaces Web portal configuration issues"
status: active
severity: MEDIUM
triggers: ["portal misconfigured", "portal settings wrong", "update portal failed", "portal INCOMPLETE"]
owner: devops-agent
objective: "Resolve portal configuration issues to ensure correct portal behavior"
context: "Portals aggregate browser settings, network settings, identity provider, trust store, user access logging, and IP access settings. Each component can be updated independently."
---

## Phase 1 — Triage

MUST:
- Get portal details: `aws workspaces-web get-portal --portal-arn <portal-arn>`
- Check portal status — must be ACTIVE for user access
- Verify associated resources: `aws workspaces-web get-browser-settings --browser-settings-arn <arn>` and `aws workspaces-web get-network-settings --network-settings-arn <arn>`

SHOULD:
- Check identity provider configuration: `aws workspaces-web list-identity-providers --portal-arn <portal-arn>`
- Verify trust store if internal sites are accessed: `aws workspaces-web list-trust-stores`

MAY:
- Review CloudTrail for recent UpdatePortal or Associate events
- Compare portal configuration against a known-good portal

## Phase 2 — Remediate

MUST:
- Associate missing resources with the portal (browser settings, network settings, IdP)
- Update portal display name and settings as needed: `aws workspaces-web update-portal --portal-arn <portal-arn> --display-name <name>`
- Verify portal status returns to ACTIVE after changes

SHOULD:
- Associate trust store for internal CA certificates: `aws workspaces-web associate-trust-store --portal-arn <portal-arn> --trust-store-arn <arn>`
- Configure user access logging for audit compliance

MAY:
- Associate IP access settings for client IP restrictions
- Configure custom session timeout values

## Safety Ratings

```
safety_ratings:
  - "get-portal: GREEN — read-only portal details"
  - "get-browser-settings: GREEN — read-only browser policy"
  - "update-portal: YELLOW — modifies portal configuration"
  - "associate-trust-store: YELLOW — changes certificate trust for sessions"
  - "associate-browser-settings: YELLOW — changes browser policies for all sessions"
```

## Escalation Conditions

- Portal status stuck in INCOMPLETE after associating all required resources
- Portal configuration changes cause all user sessions to fail
- Trust store association causes certificate validation errors for previously working sites
- Identity provider changes lock out all users

## Data Sensitivity

- `aws workspaces-web get-portal`: LOW — returns portal configuration metadata
- `aws workspaces-web get-browser-settings`: MEDIUM — reveals browser policy restrictions
- `aws workspaces-web list-identity-providers`: MEDIUM — reveals IdP configuration
- `aws workspaces-web get-network-settings`: MEDIUM — reveals VPC and subnet configuration

## Prohibited Actions

- NEVER disassociate the identity provider from a production portal without a replacement
- NEVER modify browser settings during business hours without user notification
- NEVER remove trust store certificates that are required for internal site access
- NEVER change network settings on a portal with active user sessions without a maintenance window

## Phase 3 — Rollback

1. Revert portal settings to previous configuration: `aws workspaces-web update-portal --portal-arn <portal-arn> --display-name <previous-name>`
2. Re-associate previous browser settings if they were changed
3. Re-associate previous network settings if they were changed
4. Restore trust store association if it was removed
5. Verify portal status is ACTIVE and test user access after rollback

## Common Issues

- symptoms: "Portal shows INCOMPLETE status"
  diagnosis: "One or more required associations are missing (browser settings, network settings, or IdP)."
  resolution: "Check which associations are missing and create/associate them."

## Output Format

```yaml
root_cause: "portal_configuration — <specific_cause>"
evidence:
  - type: config_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix portal configuration"
  long_term: "Document standard portal configuration and use IaC"
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
