---
title: "B3 — Document Crawl Issues"
description: "Diagnose document crawling, filtering, and processing problems"
status: active
severity: MEDIUM
triggers:
  - "documents not crawled"
  - "missing documents after sync"
  - "crawl scope wrong"
owner: devops-agent
objective: "Resolve document crawl and processing issues"
context: "Crawl scope, inclusion/exclusion patterns, and document format affect which documents are indexed."
---

## Phase 1 — Triage

MUST:
- Check sync job metrics for documents scanned vs indexed
- Review inclusion/exclusion patterns in data source config
- Verify document formats are supported
- Check for document size limits

SHOULD:
- Review crawl depth settings for web crawlers
- Check for documents exceeding size limits (50MB)
- Verify file type filters

MAY:
- Test with specific document subsets
- Check for encoding issues

## Phase 2 — Remediate

MUST:
- Adjust inclusion/exclusion patterns
- Convert unsupported document formats
- Fix crawl scope configuration

SHOULD:
- Monitor crawl metrics per sync
- Document expected document counts

## Common Issues

- symptoms: "Only 10% of expected documents indexed"
  diagnosis: "Exclusion patterns too aggressive or unsupported formats."
  resolution: "Review patterns. Check document formats. Verify crawl scope."

## Output Format

```yaml
root_cause: "crawl_issues — <specific_cause>"
evidence:
  - type: crawl_metrics
    content: "<documents scanned vs indexed>"
severity: MEDIUM
mitigation:
  immediate: "Fix crawl configuration"
  long_term: "Monitor crawl completeness"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
