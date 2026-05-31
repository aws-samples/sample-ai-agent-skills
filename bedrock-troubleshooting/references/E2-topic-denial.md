---
title: "E2 — Topic Denial Issues"
description: "Diagnose Bedrock guardrail topic denial configuration problems"
status: active
severity: MEDIUM
triggers:
  - "topic denied"
  - "topic filter"
  - "denied topic"
  - "topic blocking"
owner: devops-agent
objective: "Identify and resolve guardrail topic denial issues"
context: "Topic denial policies block conversations about specific topics. Topics are defined with descriptions and sample phrases. The model evaluates if user input or model output relates to denied topics. Overly broad topic definitions cause false positives."
---

## Phase 1 — Triage

MUST:
- Check guardrail topic policy: `aws bedrock get-guardrail --guardrail-identifier <id>`
- Review denied topic definitions and sample phrases
- Identify which topic triggered the denial
- Test with the specific content that was blocked

SHOULD:
- Check if the topic definition is too broad
- Review sample phrases for the denied topic
- Test with edge cases near the topic boundary
- Compare with allowed topics

MAY:
- Test topic denial with ApplyGuardrail API
- Review topic denial metrics

## Phase 2 — Remediate

MUST:
- Refine topic definitions to be more specific
- Update sample phrases to better represent the denied topic
- Test with representative content after changes

SHOULD:
- Use specific topic descriptions (not overly broad)
- Provide diverse sample phrases for accurate detection
- Test both positive (should block) and negative (should allow) cases

MAY:
- Create topic denial testing frameworks
- Document topic denial rationale

## Common Issues

- symptoms: "Legitimate questions blocked by topic denial"
  diagnosis: "Topic definition too broad or sample phrases too generic."
  resolution: "Narrow topic definition. Use more specific sample phrases."

- symptoms: "Topic denial not blocking intended content"
  diagnosis: "Topic definition too narrow or insufficient sample phrases."
  resolution: "Broaden topic definition. Add more diverse sample phrases."

## Output Format

```yaml
root_cause: "topic_denial — <specific_cause>"
evidence:
  - type: topic_config
    content: "<topic denial configuration>"
  - type: blocked_content
    content: "<content that triggered topic denial>"
severity: MEDIUM
mitigation:
  immediate: "Refine topic definitions"
  long_term: "Create comprehensive topic denial testing procedures"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Refine topic definitions to be more specific | YELLOW |
| Update sample phrases to better represent the denied topic | YELLOW |
| Test with representative content after changes | GREEN |
| Use specific topic descriptions (not overly broad) | GREEN |
| Provide diverse sample phrases for accurate detection | GREEN |
| Test both positive and negative cases | GREEN |
| Create topic denial testing frameworks | GREEN |
| Document topic denial rationale | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Topic denial configurations may reveal compliance requirements and restricted content categories

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If topic definitions were modified, restore the previous topic definitions in the guardrail
2. If sample phrases were changed, revert to the previous set of sample phrases
3. Publish a new guardrail version with the restored settings
4. Update the application to reference the rolled-back guardrail version
5. Verify rollback by testing with content that should be blocked and content that should be allowed

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
