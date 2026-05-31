---
title: "B2 â€” Rule-Based Matching"
description: "Diagnose rule-based matching configuration and quality issues"
status: active
severity: MEDIUM
triggers:
  - "rule-based matching error"
  - "match rules not working"
  - "too many false matches"
  - "missing matches"
owner: devops-agent
objective: "Resolve rule-based matching configuration and quality problems"
context: "Rule-based matching uses explicit rules defining attribute comparisons (EXACT, FUZZY). Within a rule, all comparisons are AND'd. Across rules, matches are OR'd. Rule order and comparison types affect match quality."
---

## Phase 1 â€” Triage

MUST:
- Get workflow rules: `aws entityresolution get-matching-workflow --workflow-name <name>`
- Review rule definitions (attributes, comparison types)
- Check match output for false positives/negatives
- Verify input data quality for matched attributes

SHOULD:
- Test with a small dataset first
- Review attribute values for inconsistencies (formatting, nulls)

## Phase 2 â€” Remediate

MUST:
- Adjust rules: tighten for fewer false positives, loosen for fewer false negatives
- Use EXACT for high-quality attributes (email), FUZZY for variable-quality (name)
- Re-run matching job after rule changes

## Common Issues

- symptoms: "Too many false positive matches"
  diagnosis: "Rules too loose â€” FUZZY on low-quality attributes."
  resolution: "Use EXACT comparison for reliable attributes. Add more attributes to rules (AND logic)."

- symptoms: "Missing expected matches"
  diagnosis: "Rules too strict or data quality issues (nulls, formatting)."
  resolution: "Add FUZZY comparison rules. Clean input data formatting. Add alternative rules (OR logic)."

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | MEDIUM — Entity Resolution processes batch matching jobs on customer data |
| Data Risk | CRITICAL — Processes PII including names, addresses, emails, and phone numbers |
| Recovery Complexity | LOW — Jobs can be re-run but consume time and compute resources |
| Blast Radius | MEDIUM — Incorrect matching can propagate bad data to downstream systems |

## Escalation Conditions

- Matching workflow producing false positives linking unrelated customer records
- Schema mapping changes causing data misalignment in production workflows
- Provider service integration failures blocking third-party identity resolution
- Job execution failures on large datasets exceeding service quotas
- Output data containing unexpected PII exposure or cross-contamination
- ML matching confidence scores dropping significantly between job runs

## Data Sensitivity

- Input datasets contain PII: names, addresses, emails, phone numbers, unique identifiers
- Match output reveals entity relationships that may be privacy-sensitive
- Schema mappings expose data structure and attribute classification
- Provider service configurations may contain third-party API credentials
- ID mapping results link identities across systems — treat as highly sensitive

## PROHIBITED Actions

- NEVER suggest deleting schema mappings that are referenced by active matching workflows
- NEVER modify matching rules on production workflows without testing on sample data first
- NEVER expose raw PII from input or output datasets in logs or troubleshooting output
- NEVER change ML matching confidence thresholds without stakeholder approval
- NEVER delete matching workflow output without confirming downstream consumers are notified
- NEVER share provider service credentials or API keys in plain text

## Phase 3 — Rollback

- If schema mapping change broke matching: revert to previous schema mapping version and re-run workflow
- If matching rules produced bad results: restore previous rule configuration and invalidate bad output
- If ML matching confidence was changed: revert threshold and re-run affected jobs
- If output configuration was modified: restore previous S3 output path and format settings
- If provider service integration broke: revert to previous provider configuration and verify connectivity
- Document all rollback actions with workflow ARN, job ID, and affected record counts
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
