---
title: "E1 — Include/Exclude Filter Issues"
description: "Diagnose DataSync include and exclude filter configuration problems"
status: active
severity: MEDIUM
triggers:
  - "filter not working"
  - "files not excluded"
  - "include filter"
  - "exclude filter"
  - "wrong files transferred"
owner: devops-agent
objective: "Identify and resolve include/exclude filter configuration issues"
context: "DataSync filters use glob patterns (not regex). Excludes are evaluated first, then includes. An empty include filter means include everything. Filters apply to relative paths from the source location subdirectory. Common issues include regex syntax (unsupported), incorrect path patterns, and filter evaluation order confusion."
---

## Phase 1 — Triage

MUST:
- Check task filter configuration: `aws datasync describe-task --task-arn <task-arn> --query '{Includes:Includes,Excludes:Excludes}'`
- Verify filters use glob syntax, NOT regex
- Check the source location subdirectory (filters are relative to this): `aws datasync describe-location-* --location-arn <source-location-arn>`
- Review which files were actually transferred: check CloudWatch Logs

SHOULD:
- Test filter patterns against expected file paths
- Verify filter evaluation order: excludes first, then includes
- Check for common glob pattern mistakes (e.g., missing leading /)

MAY:
- Run a test task with logging enabled to see which files match filters
- Compare expected vs actual file list in transfer results

## Phase 2 — Remediate

MUST:
- Use glob patterns only: *, ?, [chars] — NOT regex
- Update filters: `aws datasync update-task --task-arn <task-arn> --excludes '[{"FilterType":"SIMPLE_PATTERN","Value":"/temp/*|*.tmp"}]'`
- Include filters: `aws datasync update-task --task-arn <task-arn> --includes '[{"FilterType":"SIMPLE_PATTERN","Value":"/data/*.csv|/data/*.json"}]'`

SHOULD:
- Use | to separate multiple patterns in a single filter
- Prefix directory patterns with / for absolute paths from source root
- Test with a small dataset before running on full source

MAY:
- Use exclude filters to skip temporary, cache, or system files
- Combine include and exclude for precise file selection

## Common Issues

- symptoms: "Filter pattern not matching expected files"
  diagnosis: "Using regex syntax instead of glob patterns."
  resolution: "Replace regex with glob: *.log instead of .*\\.log$, /dir/* instead of /dir/.+"

- symptoms: "Excluded files still being transferred"
  diagnosis: "Pattern does not match the relative path from source subdirectory."
  resolution: "Verify the path is relative to the source location subdirectory, not absolute."

- symptoms: "Include filter excluding everything"
  diagnosis: "Include pattern too restrictive or path format incorrect."
  resolution: "Check pattern matches actual file paths. Use /path/*.ext format."

## Output Format

```yaml
root_cause: "filter_issue — <specific_cause>"
evidence:
  - type: filter_config
    content: "<include and exclude filter patterns>"
  - type: source_subdirectory
    content: "<source location subdirectory>"
  - type: file_paths
    content: "<sample file paths at source>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter patterns to use correct glob syntax"
  long_term: "Document filter patterns and test with small datasets first"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Filter misconfiguration may transfer unintended sensitive files |
| Blast Radius | MEDIUM — Incorrect filters transfer wrong files or miss required files |
| Rollback Complexity | LOW — Update filters and re-run task |
| Compliance Scope | MEDIUM — Filters control which data is transferred — compliance-relevant |

## Escalation Conditions

- Escalate to data owners if filters transferred unintended sensitive files
- Escalate to compliance if required files were excluded from a compliance-mandated transfer
- Escalate to operations if filter changes affect production data pipelines

## Data Sensitivity

- **Filter patterns** may reveal file naming conventions and directory structures
- **Transferred file lists** in CloudWatch Logs may contain sensitive file paths

## PROHIBITED Actions

- Do NOT deploy filter changes to production without testing on a sample dataset
- Do NOT use overly broad include filters that may capture sensitive files unintentionally
- Do NOT remove exclude filters for temporary/cache files without understanding the impact

## Phase 3 — Rollback

- Restore previous filter configuration: `aws datasync update-task --task-arn <arn> --excludes '<previous-excludes>' --includes '<previous-includes>'`
- If unintended files were transferred, delete them from the destination
- Re-run the task with corrected filters

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
