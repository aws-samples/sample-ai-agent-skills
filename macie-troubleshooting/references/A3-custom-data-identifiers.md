---
title: "A3 — Custom Data Identifiers"
description: "Diagnose custom data identifier issues in Macie"
status: active
severity: MEDIUM
triggers:
  - "custom identifier not matching"
  - "regex not working"
  - "custom pattern error"
  - "custom detection failed"
owner: devops-agent
objective: "Fix custom data identifier configuration for accurate detection"
context: "Custom data identifiers use PCRE regex (max 512 chars) with optional keywords and ignore words. They extend Macie's detection beyond managed identifiers. Issues include regex syntax errors, keyword proximity misconfiguration, and pattern specificity problems."
---

## Phase 1 — Triage

MUST:
- List custom identifiers: `aws macie2 list-custom-data-identifiers`
- Get identifier details: `aws macie2 get-custom-data-identifier --id <identifier-id>`
- Test the regex pattern: `aws macie2 test-custom-data-identifier --regex '<pattern>' --sample-text '<sample>'`
- Check if identifier is included in discovery jobs

SHOULD:
- Verify regex syntax is valid PCRE
- Check keyword proximity settings (maximumMatchDistance)
- Test with multiple sample texts including edge cases

MAY:
- Review regex for performance issues (catastrophic backtracking)
- Check if ignore words are filtering valid matches

## Phase 2 — Remediate

MUST:
- Fix regex syntax errors
- Test pattern with representative sample data
- Include identifier in discovery job configuration

SHOULD:
- Use keywords to reduce false positives (require context words near the pattern)
- Set appropriate maximumMatchDistance for keyword proximity
- Add ignore words for known false positive patterns

MAY:
- Create multiple identifiers for variations of the same data type
- Document identifier patterns and their intended use

## Common Issues

- symptoms: "Custom identifier matches nothing"
  diagnosis: "Regex pattern doesn't match the actual data format."
  resolution: "Test regex with actual sample data. Fix pattern to match real data."

- symptoms: "Custom identifier matches too much"
  diagnosis: "Regex too broad or missing keywords for context."
  resolution: "Add keywords to require context. Narrow the regex pattern."

- symptoms: "Regex timeout during scanning"
  diagnosis: "Complex regex causing catastrophic backtracking."
  resolution: "Simplify regex. Avoid nested quantifiers. Use atomic groups."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix regex pattern | GREEN | Configuration change, no data impact |
| Add keywords for context | GREEN | Improves accuracy |
| Include identifier in job | GREEN | Additive configuration |
| Test with sample data | GREEN | Read-only test operation |

## Escalation Conditions

- Custom identifier for compliance-critical data type not working — escalate to compliance team
- Regex causing scan timeouts — escalate to Macie operations team
- Custom identifier changes affecting organization-wide jobs — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-custom-data-identifier` | MEDIUM | Exposes detection pattern and keywords |
| `aws macie2 test-custom-data-identifier` | MEDIUM | Tests pattern against sample data |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** deploy untested custom identifiers to production jobs
- **NEVER** create identifiers with overly broad regex that match non-sensitive data

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed regex pattern | Restore previous regex pattern |
| Created custom identifier | Delete: `aws macie2 delete-custom-data-identifier --id <id>` |
| Added identifier to job | Remove identifier from job configuration |

## Output Format

```yaml
root_cause: "custom_identifier — <specific_cause>"
evidence:
  - type: identifier_config
    content: "<identifier regex and keywords>"
  - type: test_results
    content: "<test-custom-data-identifier results>"
severity: MEDIUM
mitigation:
  immediate: "Fix regex pattern and test with sample data"
  long_term: "Establish identifier testing and review process"
```

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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
