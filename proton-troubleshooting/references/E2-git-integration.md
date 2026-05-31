---
title: "E2 — Proton Git Integration Issues"
description: "Diagnose Git repository integration and source control problems"
status: active
severity: MEDIUM
triggers:
  - "git integration"
  - "repository access"
  - "source control"
  - "webhook"
  - "branch"
owner: devops-agent
objective: "Identify and resolve Git repository integration issues affecting Proton template sync and service pipelines"
context: "Proton integrates with Git repositories via CodeStar connections for template sync and service pipelines. Integration issues affect automatic template versioning and CI/CD pipeline source stages."
---

## Phase 1 — Triage

MUST:
- List CodeStar connections: `aws codestar-connections list-connections`
- Check connection status: `aws codestar-connections get-connection --connection-arn <connection-arn>`
- Verify repository is accessible through the connection
- Check if template sync or pipeline references the correct repository and branch
- Check CloudTrail for connection events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=codestar-connections.amazonaws.com --max-results 10`

SHOULD:
- Verify the Git provider (GitHub, Bitbucket, GitLab) OAuth app is still authorized
- Check if repository permissions have changed
- Verify branch exists and has recent commits

MAY:
- Check webhook delivery logs on the Git provider side
- Verify no IP restrictions block AWS access to the repository

## Phase 2 — Remediate

MUST:
- Re-authorize CodeStar connection if OAuth token expired
- Update connection if repository was moved or renamed
- Fix branch references in sync config or pipeline

SHOULD:
- Use dedicated service accounts for Git provider access
- Document which repositories are connected to which Proton resources
- Monitor connection health regularly

MAY:
- Create new connection if existing one is corrupted
- Set up repository access logging for audit

## Common Issues

- symptoms: "CodeStar connection in PENDING status"
  diagnosis: "Connection created but OAuth authorization not completed."
  resolution: "Complete authorization in the AWS Console — this requires manual interaction."

- symptoms: "Connection was AVAILABLE but now fails"
  diagnosis: "OAuth token expired or Git provider app was deauthorized."
  resolution: "Re-authorize the connection in the AWS Console."

- symptoms: "Pipeline source stage fails with repository error"
  diagnosis: "Repository renamed, deleted, or branch no longer exists."
  resolution: "Update the pipeline source configuration with correct repository and branch."

## Output Format

```yaml
root_cause: "git_integration_issue — <specific_cause>"
evidence:
  - type: connection_status
    content: "<CodeStar connection details>"
  - type: repository
    content: "<repository accessibility>"
  - type: cloudtrail
    content: "<connection events>"
severity: MEDIUM
mitigation:
  immediate: "Re-authorize connection or fix repository references"
  long_term: "Use dedicated service accounts and monitor connection health"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — git integration accesses source code repositories
- Service Disruption Risk: MEDIUM — integration failures prevent template sync
- Compliance Impact: MEDIUM — source code access must be properly controlled

## Escalation Conditions
- Git integration completely fails
- Repository access credentials expired
- Sync failures cause template version drift
- Unauthorized repository access detected
- Git webhook delivery failures

## Data Sensitivity
- Repository URLs — INTERNAL
- Access credentials — RESTRICTED
- Source code — CONFIDENTIAL
- Webhook configurations — INTERNAL
- Never expose repository credentials

## Prohibited Actions
- Do NOT store git credentials in plaintext
- Do NOT grant repository access to unauthorized services
- Do NOT ignore git sync failures
- Do NOT bypass repository access controls
- Do NOT share repository credentials
- Do NOT connect to untrusted repositories

## Phase 3 — Rollback
- Restore previous git integration configurations
- Rotate repository credentials if compromised
- Re-establish git connections
- Verify template sync after changes
- Document integration issues and credential management
- Update git integration procedures

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
