---
title: "B1 — Teams Integration"
description: "Diagnose Microsoft Teams integration setup failures including Azure AD, tenant consent, and workspace configuration"
status: active
severity: HIGH
triggers: ["Teams not working", "Teams integration failed", "Azure AD error", "tenant consent", "Teams workspace error"]
owner: devops-agent
objective: "Resolve Microsoft Teams integration setup and connectivity issues"
context: "Teams integration requires Azure AD app registration, tenant-level admin consent, and Chatbot workspace configuration before channel setup. This is fundamentally different from Slack integration."
---

## Phase 1 — Triage

MUST:
- List Teams workspaces: `aws chatbot list-microsoft-teams-configured-teams`
- Check workspace configuration: `aws chatbot list-microsoft-teams-channel-configurations --team-id <team-id>`
- Verify the Teams workspace is properly linked to the Azure AD tenant

SHOULD:
- Check if tenant admin consent was granted in Azure AD
- Verify the Chatbot app is installed in the Teams workspace
- Review CloudTrail for Teams configuration events

MAY:
- Check Azure AD app registration status (requires Azure portal access)
- Verify Teams channel permissions and membership

## Phase 2 — Remediate

MUST:
- Complete Azure AD app registration if not done
- Grant tenant-level admin consent for the Chatbot app
- Create Chatbot workspace configuration linking to the Teams tenant

SHOULD:
- Verify the Azure AD tenant ID matches the Chatbot workspace configuration
- Ensure the Teams admin has granted all required permissions
- Test with a non-production Teams channel first

MAY:
- Set up multiple workspace configurations for different Teams tenants
- Document the Teams integration setup for the organization

## Safety Ratings

```
safety_ratings:
  - "list-microsoft-teams-configured-teams: GREEN — read-only workspace listing"
  - "list-microsoft-teams-channel-configurations: GREEN — read-only channel listing"
  - "create-microsoft-teams-channel-configuration: YELLOW — creates new channel integration"
  - "delete-microsoft-teams-channel-configuration: YELLOW — removes channel integration"
```

## Escalation Conditions

- Azure AD admin consent requires organizational IT approval
- Tenant configuration affects multiple Teams channels and users
- Integration failure blocks critical notification delivery
- Azure AD app registration requires security review

## Data Sensitivity

- `aws chatbot list-microsoft-teams-configured-teams`: LOW — returns workspace metadata
- `aws chatbot list-microsoft-teams-channel-configurations`: LOW — returns channel configuration
- Azure AD tenant ID: MEDIUM — organizational identifier
- Teams channel membership: LOW — reveals channel participants

## Prohibited Actions

- NEVER share Azure AD tenant credentials or app secrets
- NEVER grant tenant-wide admin consent without IT security approval
- NEVER configure Teams integration with personal Microsoft accounts
- NEVER bypass Azure AD conditional access policies for Chatbot

## Phase 3 — Rollback

1. If workspace was misconfigured, delete and recreate: `aws chatbot delete-microsoft-teams-configured-team --team-id <team-id>`
2. If channel configuration is wrong, remove it: `aws chatbot delete-microsoft-teams-channel-configuration --chat-configuration-arn <arn>`
3. If Azure AD consent was granted incorrectly, revoke in Azure portal
4. Verify other Teams integrations are not affected by changes
5. Document the rollback and notify affected Teams users

## Common Issues

- symptoms: "Cannot create Teams channel configuration — workspace not found"
  diagnosis: "The Teams workspace (tenant) has not been configured in Chatbot."
  resolution: "Create the workspace configuration first: link the Azure AD tenant to Chatbot before creating channel configurations."

- symptoms: "Teams integration shows 'consent required' error"
  diagnosis: "Azure AD tenant admin has not granted consent for the Chatbot app."
  resolution: "Have a tenant admin grant consent in Azure AD for the AWS Chatbot application."

- symptoms: "Notifications not appearing in Teams channel"
  diagnosis: "The Chatbot app is not installed in the specific Teams channel, or the channel configuration is incorrect."
  resolution: "Install the Chatbot app in the Teams channel. Verify the channel ID in the Chatbot configuration matches."

## Output Format

```yaml
root_cause: "teams_integration — <specific_cause>"
evidence:
  - type: teams_error
    content: "<integration failure details>"
severity: HIGH
mitigation:
  immediate: "Fix Teams integration setup"
  long_term: "Document Teams setup process for the organization"
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
