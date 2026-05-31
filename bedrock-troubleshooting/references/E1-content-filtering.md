---
title: "E1 — Content Filtering Issues"
description: "Diagnose Bedrock guardrail content filtering problems"
status: active
severity: MEDIUM
triggers:
  - "content blocked"
  - "guardrail triggered"
  - "content filter"
  - "GUARDRAIL_INTERVENED"
owner: devops-agent
objective: "Identify and resolve guardrail content filtering issues"
context: "Bedrock guardrails filter content based on configurable thresholds for hate, insults, sexual content, violence, and misconduct. Filters apply to both input and output. Overly restrictive settings cause false positives. The guardrail response indicates which filter triggered."
---

## Phase 1 — Triage

MUST:
- Check guardrail configuration: `aws bedrock get-guardrail --guardrail-identifier <id> --guardrail-version <version>`
- Identify which filter triggered (check response for GUARDRAIL_INTERVENED)
- Review content filter thresholds (NONE, LOW, MEDIUM, HIGH)
- Determine if input or output was filtered

SHOULD:
- Test with the specific content that was blocked
- Check word filter configuration
- Review sensitive information filters (PII detection)
- Compare filter thresholds with content requirements

MAY:
- Test guardrail with ApplyGuardrail API
- Review guardrail metrics in CloudWatch

## Phase 2 — Remediate

MUST:
- Adjust content filter thresholds to appropriate levels
- Identify false positives and adjust accordingly
- Test with representative content after changes

SHOULD:
- Use DRAFT version for testing before publishing
- Document guardrail configuration rationale
- Create test suites for guardrail validation

MAY:
- Implement guardrail monitoring and alerting
- Create guardrail templates for different use cases

## Common Issues

- symptoms: "Legitimate content blocked by guardrail"
  diagnosis: "Content filter threshold too restrictive."
  resolution: "Lower the specific filter threshold. Test with representative content."

- symptoms: "Guardrail blocks input but not output (or vice versa)"
  diagnosis: "Filters apply bidirectionally but may trigger differently."
  resolution: "Check both input and output filter configurations. Adjust as needed."

## Output Format

```yaml
root_cause: "content_filtering — <specific_cause>"
evidence:
  - type: guardrail_config
    content: "<guardrail filter configuration>"
  - type: blocked_content
    content: "<content that triggered the filter>"
severity: MEDIUM
mitigation:
  immediate: "Adjust filter thresholds"
  long_term: "Create comprehensive guardrail testing procedures"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Adjust content filter thresholds to appropriate levels | RED |
| Identify false positives and adjust accordingly | YELLOW |
| Test with representative content after changes | GREEN |
| Use DRAFT version for testing before publishing | GREEN |
| Document guardrail configuration rationale | GREEN |
| Create test suites for guardrail validation | GREEN |
| Implement guardrail monitoring and alerting | GREEN |
| Create guardrail templates for different use cases | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Content that triggered filters may contain sensitive or harmful material requiring careful handling

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If guardrail thresholds were lowered, restore previous threshold values: update the guardrail and publish a new version
2. If word filters were removed, re-add them to the guardrail configuration
3. If a guardrail version was published with incorrect settings, update the application to use the previous guardrail version
4. Verify rollback by testing with the content that originally triggered the filter

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
