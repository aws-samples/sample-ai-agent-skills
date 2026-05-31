---
title: "D1 — Source Repository Issues"
description: "Diagnose source repository problems"
status: active
severity: HIGH
triggers: ["repository error", "cannot push", "clone failed", "repo not found"]
owner: devops-agent
objective: "Resolve source repository access and operation issues"
context: "CodeCatalyst supports native repos and linked repos (GitHub, Bitbucket). They have different capabilities."
---

## Phase 1 — Triage

MUST:
- List source repositories: `aws codecatalyst list-source-repositories --space-name <space> --project-name <project>`
- Get repository details: `aws codecatalyst get-source-repository --space-name <space> --project-name <project> --name <repo-name>`
- Verify user has repository access permissions

SHOULD:
- Check if the issue is with a native or linked repository
- Verify Git credentials or personal access token
- Review recent repository operations in CloudTrail

MAY:
- Test clone/push with a different user to isolate permission issues
- Check repository size and branch count

## Phase 2 — Remediate

MUST:
- Fix repository access permissions for affected users
- Regenerate personal access token if expired
- Verify Git remote URL is correct

SHOULD:
- Update linked repository connection if external repo link is broken
- Fix branch protection rules if blocking legitimate pushes
- Verify Git client version is compatible

MAY:
- Migrate from linked to native repository if link issues persist
- Configure repository webhooks for integration

## Safety Ratings

```
safety_ratings:
  - "list-source-repositories: GREEN — read-only enumeration"
  - "get-source-repository: GREEN — read-only repository details"
  - "repository permission changes: YELLOW — modifies access control"
  - "force push: RED — can overwrite repository history"
```

## Escalation Conditions

- Repository data appears corrupted
- All users unable to access the repository
- Linked repository connection permanently broken
- Repository size exceeds service limits

## Data Sensitivity

- `aws codecatalyst list-source-repositories`: LOW — returns repository names
- `aws codecatalyst get-source-repository`: LOW — returns repository metadata
- Repository content: HIGH — contains application source code
- Personal access tokens: HIGH — authentication credentials

## Prohibited Actions

- NEVER force push to shared branches without team coordination
- NEVER delete repository branches without confirming they are not in use
- NEVER share personal access tokens
- NEVER delete a repository without backing up all branches

## Phase 3 — Rollback

1. If repository settings were modified, restore original configuration
2. If branches were deleted, restore from Git reflog if possible
3. If linked repository connection was changed, restore original link
4. Verify all team members can access the repository after changes
5. Document repository changes and notify affected developers

## Common Issues

- symptoms: "Git push rejected with 'permission denied'"
  diagnosis: "User lacks push permissions or personal access token is invalid."
  resolution: "Verify user project role includes push permissions. Regenerate personal access token."

- symptoms: "Linked GitHub repository shows 'connection error'"
  diagnosis: "GitHub connection or OAuth token has expired."
  resolution: "Re-authorize the GitHub connection in CodeCatalyst space settings."

## Output Format

```yaml
root_cause: "source_repository — <specific_cause>"
evidence:
  - type: repository_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix repository access"
  long_term: "Implement repository access monitoring"
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
