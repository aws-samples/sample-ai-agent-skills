---
title: "D1 — Topic Controls Issues"
description: "Diagnose topic control blocking and configuration"
status: active
severity: MEDIUM
triggers:
  - "query blocked"
  - "topic control error"
  - "legitimate query blocked"
owner: devops-agent
objective: "Resolve topic control issues"
context: "Topic controls define allowed and blocked topics. Overly restrictive controls block legitimate queries."
---

## Phase 1 — Triage

MUST:
- Check chat controls: `aws qbusiness get-chat-controls-configuration --application-id <app-id>`
- Review blocked topic definitions
- Test with queries that should be allowed
- Check topic control descriptions

SHOULD:
- Test with diverse query formulations
- Review false positive rate
- Check for conflicting topic rules

## Phase 2 — Remediate

MUST:
- Refine topic control descriptions
- Remove overly broad blocked topics
- Test after changes

SHOULD:
- Use specific topic descriptions
- Maintain test query sets

## Common Issues

- symptoms: "Business queries blocked as off-topic"
  diagnosis: "Blocked topic description too broad."
  resolution: "Narrow blocked topic descriptions. Be specific about what to block."

## Output Format

```yaml
root_cause: "topic_controls — <specific_cause>"
evidence:
  - type: controls_config
    content: "<topic control configuration>"
severity: MEDIUM
mitigation:
  immediate: "Refine topic controls"
  long_term: "Regular testing with query sets"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — topic controls prevent unauthorized information disclosure
- Service Disruption Risk: LOW — control issues affect response filtering
- Compliance Impact: HIGH — topic controls enforce data access policies

## Escalation Conditions
- Topic controls fail to block restricted topics
- Controls too restrictive blocking legitimate queries
- Control configuration errors
- Controls bypassed by query reformulation
- Multiple control failures simultaneously

## Data Sensitivity
- Topic control configurations — CONFIDENTIAL
- Blocked topic definitions — CONFIDENTIAL
- Control evaluation logs — INTERNAL
- User query patterns — CONFIDENTIAL
- Never expose topic control configurations externally

## Prohibited Actions
- Do NOT disable topic controls on production apps
- Do NOT create overly permissive topic controls
- Do NOT bypass topic control review processes
- Do NOT ignore topic control failures
- Do NOT share topic control configurations externally
- Do NOT modify controls without compliance review

## Phase 3 — Rollback
- Restore previous topic control configurations
- Re-enable disabled controls
- Verify control effectiveness after changes
- Restore compliance-required controls
- Document control changes and compliance impact
- Update topic control management procedures

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
