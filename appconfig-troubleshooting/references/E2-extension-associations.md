---
title: "E2 — Extension Association Issues"
description: "Diagnose extension association and binding problems"
status: active
severity: MEDIUM
triggers:
  - "extension not running"
  - "association error"
  - "extension not triggered"
owner: devops-agent
objective: "Correctly associate extensions with AppConfig resources"
context: "Extensions must be associated with specific resources (applications, environments, or configuration profiles) to execute."
---

## Phase 1 — Triage

MUST:
- List associations: `aws appconfig list-extension-associations --resource-identifier <arn>`
- Verify the extension is associated with the correct resource level
- Check if the extension version matches the association

SHOULD:
- Verify the action point matches when the extension should run
- Check if multiple extensions conflict at the same action point

MAY:
- Review CloudTrail for association creation/deletion events

## Phase 2 — Remediate

MUST:
- Create the missing association if extension is not running
- Fix the resource ARN if associated with wrong resource

SHOULD:
- Associate at the most specific level needed (profile > environment > application)

MAY:
- Document extension associations for operational visibility

safety_ratings:
  - "Listing associations: GREEN"
  - "Creating association: GREEN"
  - "Deleting association: YELLOW"

escalation_conditions:
  - "Security extension not associated with production resources"

data_sensitivity:
  - command: "list-extension-associations"
    sensitivity: LOW
    contains: "Resource ARNs, extension IDs"

prohibited_actions:
  - "NEVER remove security or compliance extension associations without approval"

## Phase 3 — Rollback

- If association was removed, recreate it with the same parameters
- If association was added incorrectly, delete it

## Common Issues

- symptoms: "Extension exists but never executes"
  diagnosis: "Extension not associated with the target resource"
  resolution: "Create an extension association with the correct resource ARN"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: association_config
    content: "<association details>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
