---
title: "A1 — Code Suggestions Not Working"
description: "Diagnose missing or non-functional code suggestions"
status: active
severity: HIGH
triggers:
  - "no suggestions"
  - "suggestions not appearing"
  - "Q Developer not working"
owner: devops-agent
objective: "Resolve code suggestion failures"
context: "Code suggestions require authentication, network connectivity, and sufficient code context."
---

## Phase 1 — Triage

MUST:
- Verify authentication status in IDE (Builder ID or Identity Center)
- Check IDE plugin is installed and enabled
- Verify network connectivity to AWS endpoints
- Check if free tier limits are exceeded

SHOULD:
- Check IDE plugin version (update if outdated)
- Verify file type is supported
- Check for proxy/firewall blocking
- Review IDE plugin logs

MAY:
- Test with a simple file to isolate issue
- Check for conflicting IDE extensions

## Phase 2 — Remediate

MUST:
- Sign in if not authenticated
- Update plugin if outdated
- Fix network connectivity

SHOULD:
- Configure proxy settings if behind corporate firewall
- Disable conflicting extensions

## Common Issues

- symptoms: "Plugin installed but no suggestions appear"
  diagnosis: "Not signed in or authentication expired."
  resolution: "Sign in via AWS Builder ID or IAM Identity Center."

- symptoms: "Suggestions stopped mid-month"
  diagnosis: "Free tier monthly limit exceeded."
  resolution: "Wait for next month or upgrade to Pro tier."

## Output Format

```yaml
root_cause: "code_suggestions — <specific_cause>"
evidence:
  - type: plugin_status
    content: "<authentication and plugin status>"
severity: HIGH
mitigation:
  immediate: "Fix authentication or connectivity"
  long_term: "Monitor usage limits"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — code suggestions may reference proprietary code patterns
- Service Disruption Risk: LOW — suggestion issues do not affect development workflow
- Compliance Impact: MEDIUM — code suggestions must comply with code reference policies

## Escalation Conditions
- Code suggestions completely stop working
- Suggestions contain inappropriate or insecure code patterns
- Suggestion latency exceeds acceptable thresholds
- Suggestions reference unauthorized code sources
- Suggestion quality degrades significantly

## Data Sensitivity
- Code context sent for suggestions — CONFIDENTIAL
- Suggestion results — INTERNAL
- User coding patterns — INTERNAL
- IDE telemetry — INTERNAL
- Never expose proprietary code context externally

## Prohibited Actions
- Do NOT accept code suggestions without review
- Do NOT disable code suggestion security scanning
- Do NOT share suggestion telemetry externally
- Do NOT use suggestions containing known security vulnerabilities
- Do NOT bypass code review for AI-generated suggestions
- Do NOT ignore suggestion quality degradation

## Phase 3 — Rollback
- Restore previous suggestion configurations
- Revert suggestion preference changes
- Re-enable suggestion features if disabled
- Verify suggestion functionality after changes
- Document suggestion issues and quality metrics
- Update suggestion usage guidelines

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
