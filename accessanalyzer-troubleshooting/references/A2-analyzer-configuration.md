---
title: "A2 — Analyzer Configuration"
description: "Diagnose analyzer configuration issues"
status: active
severity: MEDIUM
triggers: ["analyzer not working", "analyzer status", "analyzer config"]
owner: devops-agent
objective: "Resolve analyzer configuration issues"
context: "Analyzers must be in ACTIVE status to generate findings. Configuration includes type, archive rules, and unused access settings."
---

## Phase 1 — Triage

MUST:
- Check analyzer status: `aws accessanalyzer get-analyzer --analyzer-name <name>`
- Verify status is ACTIVE (not CREATING or FAILED)
- Check analyzer type matches intended scope

SHOULD:
- Review archive rules: `aws accessanalyzer list-archive-rules --analyzer-name <name>`
- Check unused access configuration if applicable

MAY:
- Compare with analyzers in other regions

## Phase 2 — Remediate

MUST:
- If FAILED: delete and recreate analyzer
- Verify analyzer type is correct for use case

SHOULD:
- Configure archive rules for known acceptable findings

MAY:
- Set up EventBridge rules for finding notifications

## Common Issues

- symptoms: "Analyzer shows CREATING status for hours"
  diagnosis: "Initial analysis of all resources takes time for large accounts."
  resolution: "Wait for analysis to complete. Check status periodically."

## Safety Ratings

```
safety_ratings:
  - "get-analyzer: GREEN — read-only status check"
  - "list-archive-rules: GREEN — read-only enumeration"
  - "delete and recreate analyzer: RED — destroys existing findings and archive rules"
  - "EventBridge rule creation: YELLOW — adds automation that may trigger actions"
```

## Escalation Conditions

- Analyzer stuck in CREATING or FAILED status for more than 1 hour
- Analyzer type mismatch requires deletion and recreation (findings will be lost)
- Archive rules are suspected of hiding critical security findings
- Analyzer configuration changes needed in a production organization account

## Data Sensitivity

- `aws accessanalyzer get-analyzer`: LOW — returns analyzer metadata
- `aws accessanalyzer list-archive-rules`: MEDIUM — reveals what findings are being suppressed
- Analyzer deletion: HIGH — permanently destroys all associated findings

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER delete and recreate an analyzer without first exporting or documenting existing findings
- NEVER modify archive rules without reviewing what findings they currently suppress

## Phase 3 — Rollback

1. If archive rules were modified incorrectly, restore previous rules: `aws accessanalyzer update-archive-rule --analyzer-name <name> --rule-name <rule> --filter <original-filter>`
2. If an analyzer was deleted, recreate it: `aws accessanalyzer create-analyzer --analyzer-name <name> --type <ACCOUNT|ORGANIZATION>` (note: previous findings are lost)
3. If EventBridge rules were created in error, disable or delete them: `aws events disable-rule --name <rule-name>`
4. Verify analyzer returns to ACTIVE status after remediation
5. Confirm findings are being generated for known external access patterns

## Output Format

```yaml
root_cause: "analyzer_configuration — <specific_cause>"
evidence:
  - type: analyzer_status
    content: "<analyzer status and configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration"
  long_term: "Monitor analyzer health"
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
