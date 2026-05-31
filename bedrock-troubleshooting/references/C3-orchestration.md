---
title: "C3 — Agent Orchestration Issues"
description: "Diagnose Bedrock agent orchestration loops and failures"
status: active
severity: MEDIUM
triggers:
  - "orchestration loop"
  - "agent timeout"
  - "agent not completing"
  - "orchestration trace"
owner: devops-agent
objective: "Identify and resolve agent orchestration issues"
context: "Agents use multi-step orchestration to break down tasks, invoke action groups, and synthesize responses. Orchestration can loop if the agent cannot determine the next step, action group responses are ambiguous, or the instruction prompt is unclear. Max iterations and timeout control behavior."
---

## Phase 1 — Triage

MUST:
- Enable trace in agent invocation to see orchestration steps
- Check for orchestration loops (repeated action group calls)
- Review agent instruction prompt for clarity
- Check action group descriptions for ambiguity

SHOULD:
- Analyze the orchestration trace for decision points
- Check if the agent is receiving expected action group responses
- Verify max iterations setting
- Test with simpler prompts to isolate the issue

MAY:
- Compare orchestration behavior across different models
- Review session history for context issues

## Phase 2 — Remediate

MUST:
- Clarify agent instruction prompt with specific guidelines
- Improve action group descriptions for better routing
- Set appropriate max iterations to prevent infinite loops

SHOULD:
- Add examples to the instruction prompt
- Ensure action group responses are clear and actionable
- Test orchestration with trace enabled

MAY:
- Implement orchestration monitoring
- Create prompt engineering guidelines for agents

## Common Issues

- symptoms: "Agent loops between action groups"
  diagnosis: "Ambiguous action group descriptions or unclear instruction prompt."
  resolution: "Clarify action group descriptions. Add routing guidance to instruction prompt."

- symptoms: "Agent times out before completing"
  diagnosis: "Too many orchestration steps or slow action groups."
  resolution: "Simplify the task. Optimize action group Lambda functions. Increase timeout."

## Output Format

```yaml
root_cause: "orchestration — <specific_cause>"
evidence:
  - type: orchestration_trace
    content: "<orchestration steps and decisions>"
  - type: agent_config
    content: "<instruction prompt and action groups>"
severity: MEDIUM
mitigation:
  immediate: "Fix instruction prompt and action group descriptions"
  long_term: "Implement orchestration monitoring and prompt engineering best practices"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Clarify agent instruction prompt with specific guidelines | GREEN |
| Improve action group descriptions for better routing | GREEN |
| Set appropriate max iterations to prevent infinite loops | GREEN |
| Add examples to the instruction prompt | GREEN |
| Ensure action group responses are clear and actionable | GREEN |
| Test orchestration with trace enabled | GREEN |
| Implement orchestration monitoring | GREEN |
| Create prompt engineering guidelines for agents | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Orchestration traces may reveal sensitive data flowing through agent interactions

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If the instruction prompt was changed, restore the previous prompt text
2. If action group descriptions were modified, revert to previous descriptions
3. If max iterations was changed, restore the previous value
4. Run `aws bedrock-agent prepare-agent --agent-id <id>` after reverting changes
5. Verify rollback by testing orchestration with trace enabled and confirming expected behavior

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
