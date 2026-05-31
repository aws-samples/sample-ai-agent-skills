---
title: "C1 — Agent Creation Issues"
description: "Diagnose Bedrock agent creation and configuration failures"
status: active
severity: HIGH
triggers:
  - "agent creation failed"
  - "CreateAgent error"
  - "agent configuration"
  - "agent not working"
owner: devops-agent
objective: "Identify and resolve Bedrock agent creation issues"
context: "Agent creation requires a foundation model, IAM role, instruction prompt, and optionally action groups and knowledge bases. Failures stem from model access, role permissions, invalid configuration, or action group setup issues."
---

## Phase 1 — Triage

MUST:
- Check agent status: `aws bedrock-agent get-agent --agent-id <id>`
- Verify the foundation model is accessible for the agent
- Check agent execution role permissions
- Verify agent instruction prompt is configured

SHOULD:
- Check action group configurations: `aws bedrock-agent list-agent-action-groups --agent-id <id> --agent-version DRAFT`
- Verify knowledge base associations
- Check CloudTrail for CreateAgent events
- Test agent with a simple prompt

MAY:
- Review agent instruction prompt for clarity
- Check for agent version issues

## Phase 2 — Remediate

MUST:
- Ensure foundation model access is approved
- Create IAM role with bedrock:InvokeModel and action group permissions
- Configure clear instruction prompt

SHOULD:
- Start with a simple agent (no action groups) and add complexity
- Test with DRAFT version before publishing
- Prepare the agent after configuration changes: `aws bedrock-agent prepare-agent --agent-id <id>`

MAY:
- Create agent templates for common use cases
- Set up agent monitoring

## Common Issues

- symptoms: "Agent creation fails with model access error"
  diagnosis: "Foundation model not approved for the account/region."
  resolution: "Request model access in the Bedrock console."

- symptoms: "Agent created but not responding"
  diagnosis: "Agent not prepared after configuration changes."
  resolution: "Run prepare-agent to apply configuration changes."

## Output Format

```yaml
root_cause: "agent_creation — <specific_cause>"
evidence:
  - type: agent_status
    content: "<agent configuration and status>"
  - type: model_access
    content: "<foundation model access status>"
severity: HIGH
mitigation:
  immediate: "Fix agent configuration and prepare"
  long_term: "Create agent templates and testing procedures"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure foundation model access is approved | GREEN |
| Create IAM role with bedrock:InvokeModel and action group permissions | YELLOW |
| Configure clear instruction prompt | GREEN |
| Start with a simple agent (no action groups) and add complexity | GREEN |
| Test with DRAFT version before publishing | GREEN |
| Prepare the agent after configuration changes | GREEN |
| Create agent templates for common use cases | GREEN |
| Set up agent monitoring | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Agent instruction prompts and action group configurations may contain business logic

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If an agent was created incorrectly, delete it: `aws bedrock-agent delete-agent --agent-id <id>`
2. If IAM roles were modified, revert to the previous policy version
3. If the agent instruction prompt was changed, restore the previous prompt and run `aws bedrock-agent prepare-agent --agent-id <id>`
4. If action groups were misconfigured, remove them: `aws bedrock-agent delete-agent-action-group --agent-id <id> --action-group-id <ag-id>`
5. Verify rollback by testing the agent with a known prompt

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
