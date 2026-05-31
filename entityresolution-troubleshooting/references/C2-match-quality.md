---
title: "C2 â€” Match Quality"
description: "Diagnose and improve matching quality and accuracy"
status: active
severity: MEDIUM
triggers:
  - "poor match quality"
  - "false positives"
  - "false negatives"
  - "match accuracy"
owner: devops-agent
objective: "Improve matching quality for both rule-based and ML matching"
context: "Match quality depends on input data quality, attribute mapping, matching rules/model, and confidence thresholds. Both false positives (incorrect matches) and false negatives (missed matches) affect quality."
---

## Phase 1 â€” Triage

MUST:
- Review match output: check matched pairs and confidence scores
- Analyze false positives: identify incorrectly matched records
- Analyze false negatives: identify missed matches
- Check input data quality (nulls, formatting inconsistencies)

SHOULD:
- Sample and manually review match results
- Compare match rates across different attribute combinations
- Check for data quality issues in source tables

## Phase 2 â€” Remediate

MUST:
- For false positives: tighten rules or increase confidence threshold
- For false negatives: loosen rules, add FUZZY matching, or lower threshold
- Improve input data quality (standardize formats, fill nulls)

## Common Issues

- symptoms: "High false positive rate"
  diagnosis: "Matching on common values (e.g., common names without additional attributes)."
  resolution: "Add more discriminating attributes to rules. Use EXACT on unique attributes."

- symptoms: "High false negative rate"
  diagnosis: "Data quality issues or overly strict matching rules."
  resolution: "Standardize input data. Add FUZZY rules. Create alternative rule combinations."

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
