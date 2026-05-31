---
title: "F2 — Proton CodeStar Connection Issues"
description: "Diagnose CodeStar connection problems affecting Proton Git integration"
status: active
severity: MEDIUM
triggers:
  - "CodeStar connection"
  - "connection pending"
  - "connection failed"
  - "repository access"
  - "OAuth"
owner: devops-agent
objective: "Identify and resolve CodeStar connection issues affecting Proton template sync and pipelines"
context: "CodeStar connections provide Git repository access for Proton template sync and service pipelines. Connections require manual OAuth authorization and can fail due to expired tokens, revoked access, or provider issues."
---

## Phase 1 — Triage

MUST:
- List all connections: `aws codestar-connections list-connections`
- Check specific connection: `aws codestar-connections get-connection --connection-arn <connection-arn>`
- Verify connection status is AVAILABLE (not PENDING or ERROR)
- Check which Proton resources use this connection
- Check CloudTrail for connection events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codestar-connections.amazonaws.com --max-results 10`

SHOULD:
- Verify the Git provider (GitHub, Bitbucket) app is still installed and authorized
- Check if the connection owner still has repository access
- Verify no organization-level restrictions block the connection

MAY:
- Check connection tags for ownership information
- Review connection creation date and last used time

## Phase 2 — Remediate

MUST:
- Authorize PENDING connections through the AWS Console (cannot be done via CLI)
- Re-authorize connections with expired OAuth tokens
- Create new connection if existing one is permanently broken

SHOULD:
- Use organization-level Git provider apps for broader access
- Document connection ownership and purpose
- Monitor connection health regularly

MAY:
- Set up CloudWatch Events for connection status changes
- Implement connection rotation procedures

## Common Issues

- symptoms: "Connection status is PENDING"
  diagnosis: "Connection created via API/CLI but OAuth authorization not completed."
  resolution: "Go to AWS Console → CodeStar Connections → complete the authorization flow."

- symptoms: "Connection was AVAILABLE but template sync stopped"
  diagnosis: "OAuth token expired or Git provider app was uninstalled."
  resolution: "Re-authorize the connection in the AWS Console."

- symptoms: "Connection AVAILABLE but repository not accessible"
  diagnosis: "Repository permissions changed or repository was moved/deleted."
  resolution: "Verify repository exists and the connection has access. May need new connection."

## Output Format

```yaml
root_cause: "codestar_connection_issue — <specific_cause>"
evidence:
  - type: connection_status
    content: "<connection state and details>"
  - type: provider_status
    content: "<Git provider app authorization>"
  - type: cloudtrail
    content: "<connection events>"
severity: MEDIUM
mitigation:
  immediate: "Authorize or re-authorize the connection"
  long_term: "Monitor connection health and implement rotation procedures"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — CodeStar connections access source code repositories
- Service Disruption Risk: MEDIUM — connection failures prevent template sync
- Compliance Impact: MEDIUM — source code access must be properly controlled

## Escalation Conditions
- CodeStar connection becomes unavailable
- Connection authorization expires
- Connection permissions insufficient for required operations
- Multiple connections fail simultaneously
- Connection security compromised

## Data Sensitivity
- Connection configurations — INTERNAL
- Repository access details — CONFIDENTIAL
- Authorization tokens — RESTRICTED
- Connection status data — INTERNAL
- Never expose connection credentials

## Prohibited Actions
- Do NOT share CodeStar connection credentials
- Do NOT grant connections access to unauthorized repositories
- Do NOT ignore connection authorization expiration
- Do NOT bypass connection security review
- Do NOT create connections to untrusted providers
- Do NOT store connection tokens in plaintext

## Phase 3 — Rollback
- Restore previous connection configurations
- Re-authorize expired connections
- Rotate connection credentials if compromised
- Verify template sync after connection changes
- Document connection issues and authorization status
- Update connection management procedures

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
