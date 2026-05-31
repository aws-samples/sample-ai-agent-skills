---
title: "C1 — Rule-Based Matching Tuning"
description: "Diagnose and tune rule-based matching configuration for accuracy and performance"
status: active
severity: MEDIUM
triggers: ["matching rules not working", "too many false positives", "too many false negatives", "rule tuning", "matching accuracy"]
owner: devops-agent
objective: "Optimize rule-based matching configuration for desired accuracy"
context: "Rule-based matching uses ordered rules evaluated sequentially. Rule order, field selection, and comparison types affect matching quality. More specific rules should precede broader rules."
---

## Phase 1 — Triage

MUST:
- Get workflow configuration with rules: `aws entityresolution get-matching-workflow --workflow-name <workflow-name>`
- Review the rule definitions, order, and field comparisons
- Check recent job results for match rates and quality

SHOULD:
- Sample matched and unmatched records to assess accuracy
- Identify false positives (incorrectly matched) and false negatives (missed matches)
- Review the schema mapping to ensure correct fields are available for rules

MAY:
- Compare results across different rule configurations
- Analyze data distribution for fields used in rules

## Phase 2 — Remediate

MUST:
- Reorder rules: most specific first, broadest last
- Fix field comparison types (exact match, fuzzy match, etc.)
- Ensure rules reference fields that are populated in the data

SHOULD:
- Add rules for common matching patterns in the data
- Remove or modify rules that cause excessive false positives
- Test rule changes with a representative data sample

MAY:
- Implement a multi-pass matching strategy (strict rules first, then relaxed)
- Create separate workflows for different data quality tiers

## Safety Ratings

```
safety_ratings:
  - "get-matching-workflow: GREEN — read-only workflow configuration"
  - "update-matching-workflow: YELLOW — modifies matching rules"
  - "start-matching-job: YELLOW — runs matching with updated rules"
  - "delete-matching-workflow: RED — permanently removes workflow"
```

## Escalation Conditions

- Rule changes significantly affect match rates in production
- False positive matches cause downstream data quality issues
- Rule complexity exceeds manageable levels
- Business requirements conflict with achievable matching accuracy

## Data Sensitivity

- `aws entityresolution get-matching-workflow`: LOW — workflow configuration
- Matched records: HIGH — contains PII from matched entities
- Unmatched records: HIGH — contains PII
- Rule configuration: LOW — contains matching logic

## Prohibited Actions

- NEVER deploy rule changes to production without testing on sample data
- NEVER remove all rules to start over without backing up the configuration
- NEVER use overly broad rules (e.g., match on single common field) without additional constraints
- NEVER ignore false positive rates — they indicate rule quality issues

## Phase 3 — Rollback

1. If rule changes degraded matching quality, restore previous rule configuration
2. Re-run the matching job with the restored rules
3. Compare results to verify rollback improved quality
4. Document the rule change attempt and results
5. Plan a more careful tuning approach

## Common Issues

- symptoms: "Too many false positives — unrelated records matched together"
  diagnosis: "Rules are too broad or match on fields with low cardinality (e.g., city, state)."
  resolution: "Add more specific fields to rules. Require multiple field matches. Use exact match for high-cardinality fields."

- symptoms: "Too many false negatives — known matches not found"
  diagnosis: "Rules are too strict or data has inconsistent formatting (e.g., 'St.' vs 'Street')."
  resolution: "Use fuzzy matching for fields with formatting variations. Add alternative rules for common patterns."

- symptoms: "Rule order causes incorrect match grouping"
  diagnosis: "A broad rule matches records before a more specific rule can evaluate them."
  resolution: "Reorder rules: most specific first. Records matched by earlier rules are not re-evaluated by later rules."

## Output Format

```yaml
root_cause: "rule_matching_tuning — <specific_cause>"
evidence:
  - type: matching_quality
    content: "<match rate, false positive/negative analysis>"
severity: MEDIUM
mitigation:
  immediate: "Adjust matching rules"
  long_term: "Implement matching quality monitoring and validation"
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
