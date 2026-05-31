---
title: "C2 — Collaboration Issues"
description: "Diagnose WorkDocs collaboration and co-editing issues"
status: active
severity: MEDIUM
triggers: ["cannot comment", "version conflict", "document locked", "feedback not working"]
owner: devops-agent
objective: "Resolve collaboration issues including commenting, versioning, document locking, and feedback workflows"
context: "WorkDocs supports commenting, version control, and feedback requests. Document locking prevents concurrent edits."
---

## Phase 1 — Triage

MUST:
- Check document details and lock status: `aws workdocs get-document --document-id <doc-id>`
- Check document versions: `aws workdocs describe-document-versions --document-id <doc-id>`
- Verify user has CONTRIBUTOR or higher permission: `aws workdocs describe-resource-permissions --resource-id <doc-id>`
- Check if the document is in a feedback cycle

SHOULD:
- Verify the user's account type supports commenting (MINIMALUSER cannot comment)
- Check if the document format supports online editing
- Review recent activity on the document: `aws workdocs describe-activities --resource-id <doc-id>`

MAY:
- Check if there are pending feedback requests
- Verify notification settings for collaboration events

## Phase 2 — Remediate

MUST:
- Unlock documents that are incorrectly locked (admin action)
- Grant CONTRIBUTOR permission if user needs to comment or edit
- Resolve version conflicts by identifying the correct version

SHOULD:
- Upgrade MINIMALUSER accounts if they need collaboration features
- Clear stale document locks from disconnected sessions

MAY:
- Set up feedback workflows for document review processes
- Configure notification preferences for collaboration events

## Safety Ratings

```
safety_ratings:
  - "workdocs get-document: GREEN — read-only document inspection"
  - "workdocs describe-document-versions: GREEN — read-only version listing"
  - "workdocs describe-resource-permissions: GREEN — read-only permission check"
  - "workdocs update-document: YELLOW — modifies document metadata or lock status"
```

## Escalation Conditions

- Document is locked by a user who is no longer active or available
- Version history shows data corruption or missing versions
- Collaboration features are non-functional for all users
- Feedback workflow is stuck and cannot be completed or cancelled

## Data Sensitivity

- `aws workdocs get-document`: MEDIUM — reveals document metadata and owner
- `aws workdocs describe-document-versions`: MEDIUM — reveals version history and editors
- `aws workdocs describe-activities`: MEDIUM — reveals user collaboration patterns

## Prohibited Actions

- NEVER delete document versions to resolve conflicts — versions are audit trail
- NEVER force-unlock documents without contacting the locking user first
- NEVER modify document content directly to resolve version conflicts
- NEVER disable collaboration features organization-wide to fix individual issues

## Phase 3 — Rollback

1. If a document was unlocked, notify the original locking user
2. If document metadata was modified, restore original settings
3. If permissions were changed for collaboration, revert unauthorized changes
4. If a feedback cycle was cancelled, notify all participants
5. Document the collaboration issue and resolution for team reference

## Common Issues

- symptoms: "Cannot edit document — shows as locked by another user"
  diagnosis: "Document is locked by a user who may have disconnected without saving."
  resolution: "Contact the locking user. If unavailable, admin can unlock: update-document with lock removal."

- symptoms: "Comments not visible to other users"
  diagnosis: "Comments may be on a different document version, or user lacks permission to view."
  resolution: "Verify all users are viewing the latest version. Check permissions include VIEWER or higher."

## Output Format

```yaml
root_cause: "collaboration — <specific_cause>"
evidence:
  - type: collaboration_issue
    content: "<document status, lock info, version details>"
severity: MEDIUM
mitigation:
  immediate: "Resolve collaboration blocker"
  long_term: "Establish document collaboration guidelines"
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
