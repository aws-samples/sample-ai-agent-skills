---
title: "D2 — Pull Request Problems"
description: "Diagnose pull request and code review issues"
status: active
severity: MEDIUM
triggers: ["PR failed", "cannot merge", "review error", "pull request stuck"]
owner: devops-agent
objective: "Resolve pull request creation, review, and merge issues"
context: "CodeCatalyst has its own PR system. While similar to GitHub PRs, the workflow and features differ."
---

## Phase 1 — Triage

MUST:
- List pull requests: `aws codecatalyst list-source-repository-branches --space-name <space> --project-name <project> --source-repository-name <repo>`
- Verify the source and destination branches exist
- Check if merge conflicts exist

SHOULD:
- Review PR approval requirements and status
- Check if required workflow checks have passed
- Verify the user has permission to create/merge PRs

MAY:
- Review PR comments for reviewer feedback
- Check if branch protection rules are blocking the merge

## Phase 2 — Remediate

MUST:
- Resolve merge conflicts in the source branch
- Obtain required approvals from designated reviewers
- Ensure all required workflow checks pass

SHOULD:
- Update the source branch with latest changes from destination
- Address reviewer comments before requesting re-review
- Verify the merge strategy is appropriate (merge, squash)

MAY:
- Configure auto-merge for PRs that pass all checks
- Set up PR templates for consistent descriptions

## Safety Ratings

```
safety_ratings:
  - "list-source-repository-branches: GREEN — read-only branch listing"
  - "PR creation: GREEN — creates review request"
  - "PR merge: YELLOW — modifies destination branch"
  - "force merge: RED — bypasses review requirements"
```

## Escalation Conditions

- PR merge blocked by system error (not policy)
- Required reviewers are unavailable for extended period
- Merge conflicts cannot be resolved without significant refactoring
- PR blocks critical production deployment

## Data Sensitivity

- PR metadata: LOW — branch names and review status
- PR diff content: HIGH — contains code changes
- Reviewer comments: MEDIUM — may contain security-relevant feedback
- Merge history: LOW — branch merge records

## Prohibited Actions

- NEVER force merge PRs bypassing required reviews
- NEVER approve your own pull requests
- NEVER merge PRs with failing security checks
- NEVER delete source branches before confirming merge is complete

## Phase 3 — Rollback

1. If a PR was merged incorrectly, revert the merge commit on the destination branch
2. If branch protection rules were modified, restore original rules
3. Verify the destination branch is in a clean state after rollback
4. Confirm workflows triggered by the merge are cancelled or rolled back
5. Document the PR issue and notify affected team members

## Common Issues

- symptoms: "Cannot merge PR — 'required checks not passed'"
  diagnosis: "Workflow checks triggered by the PR have not completed or have failed."
  resolution: "Review workflow run status. Fix failing checks and re-run workflows."

- symptoms: "PR shows merge conflicts that cannot be resolved in UI"
  diagnosis: "Complex conflicts require local resolution."
  resolution: "Pull both branches locally, resolve conflicts, push the resolved source branch."

## Output Format

```yaml
root_cause: "pull_request — <specific_cause>"
evidence:
  - type: pr_status
    content: "<PR details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve PR blocker"
  long_term: "Configure PR policies and automation"
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
