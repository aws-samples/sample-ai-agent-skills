---
title: "G2 — Archive Management"
description: "Diagnose archive management issues"
status: active
severity: LOW
triggers: ["archived findings", "unarchive", "archive management"]
owner: devops-agent
objective: "Resolve archive management issues"
context: "Archived findings can be viewed and unarchived. Archive rules can be updated or deleted."
---

## Phase 1 — Triage

MUST:
- List archived findings: `aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"status":{"eq":["ARCHIVED"]}}'`
- Review archive rules: `aws accessanalyzer list-archive-rules --analyzer-name <name>`

SHOULD:
- Check if important findings were accidentally archived
- Review archive rule filters for over-matching

MAY:
- Audit archive history

## Phase 2 — Remediate

MUST:
- Unarchive incorrectly archived findings: `aws accessanalyzer update-findings --analyzer-arn <arn> --ids <id> --status ACTIVE`
- Fix or delete over-broad archive rules

SHOULD:
- Review archive rules periodically
- Narrow over-broad filters

MAY:
- Set up alerts for newly archived findings

## Common Issues

- symptoms: "Important finding was auto-archived by rule"
  diagnosis: "Archive rule filter too broad."
  resolution: "Unarchive the finding. Narrow the archive rule filter."

## Safety Ratings

```
safety_ratings:
  - "list-findings (ARCHIVED filter): GREEN — read-only enumeration"
  - "list-archive-rules: GREEN — read-only enumeration"
  - "update-findings to ACTIVE: YELLOW — restores finding visibility, may trigger alerts"
  - "delete archive rule: YELLOW — future findings will no longer be auto-archived"
```

## Escalation Conditions

- Important security findings were auto-archived by an overly broad rule
- Archive rules are being used to hide findings instead of resolving underlying issues
- Audit reveals archived findings that should have been investigated
- Archive rule changes affect findings across the entire organization analyzer

## Data Sensitivity

- `aws accessanalyzer list-findings (ARCHIVED)`: MEDIUM — reveals suppressed access patterns
- Archive rule filters: MEDIUM — reveals organizational risk acceptance criteria
- Unarchiving findings: MEDIUM — may trigger downstream alerts in Security Hub

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER delete all archive rules without understanding the impact on finding volume
- NEVER use archive management as a substitute for fixing underlying access issues

## Phase 3 — Rollback

1. If findings were unarchived incorrectly, re-archive them: `aws accessanalyzer update-findings --analyzer-arn <arn> --ids <finding-ids> --status ARCHIVED`
2. If an archive rule was deleted, recreate it: `aws accessanalyzer create-archive-rule --analyzer-name <name> --rule-name <rule> --filter <filter>`
3. If archive rule filters were narrowed too aggressively, restore the original filter scope
4. Verify the finding management workflow is functioning as expected
5. Confirm no critical findings are being suppressed after changes

## Output Format

```yaml
root_cause: "archive_management — <specific_cause>"
evidence:
  - type: archive_status
    content: "<archived findings and rules>"
severity: LOW
mitigation:
  immediate: "Fix archive rules"
  long_term: "Regular archive rule review"
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
