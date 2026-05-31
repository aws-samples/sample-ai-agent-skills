---
title: "Z1 — General Bedrock Troubleshooting"
description: "Catch-all systematic investigation for Bedrock issues"
status: active
severity: MEDIUM
triggers:
  - "Bedrock issue"
  - "Bedrock error"
  - "Bedrock problem"
  - "Bedrock not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Bedrock issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Bedrock issues that don't clearly match a specific runbook. It covers model access, invocation testing, KB status, agent status, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- Check model access: verify models are approved in the Bedrock console
- Test basic invocation: `aws bedrock-runtime invoke-model --model-id <model-id> --body '<minimal-request>' output.json`
- List knowledge bases: `aws bedrock-agent list-knowledge-bases`
- List agents: `aws bedrock-agent list-agents`

SHOULD:
- Check CloudTrail for recent Bedrock events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock.amazonaws.com --max-results 20`
- Check CloudWatch metrics for errors and throttling
- Verify IAM permissions
- Check for guardrail interference

MAY:
- Check AWS Health Dashboard for Bedrock service issues
- Review application logs for error details

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Model issues → A1 (access), A2 (invocation), A3 (throttling)
  - Knowledge base → B1 (creation), B2 (sync), B3 (retrieval)
  - Agents → C1 (creation), C2 (action groups), C3 (orchestration)
  - Fine-tuning → D1 (training), D2 (provisioned throughput)
  - Guardrails → E1 (content filtering), E2 (topic denial)
  - Performance → F1 (latency), F2 (token limits)
  - Security → G1 (IAM), G2 (VPC)
  - Integration → H1 (Lambda), H2 (streaming)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Common Issues

- symptoms: "Bedrock calls failing"
  diagnosis: "Multiple possible causes: model access, IAM, throttling, format."
  resolution: "Check model access → IAM permissions → request format → throttling."

- symptoms: "Bedrock was working but stopped"
  diagnosis: "Model deprecation, IAM change, or rate limit change."
  resolution: "Check model lifecycle status. Review IAM changes. Check throttling metrics."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: model_access
    content: "<model access status>"
  - type: invocation_test
    content: "<basic invocation result>"
  - type: services
    content: "<KB and agent status>"
severity: MEDIUM
general_analysis:
  model_access: "approved | not approved"
  invocation: "working | failing"
  knowledge_bases: "healthy | issues"
  agents: "healthy | issues"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive Bedrock monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Pivot to the appropriate specific runbook based on triage findings | GREEN |
| Document findings and root cause | GREEN |
| Check for multiple concurrent issues | GREEN |
| Review recent changes | GREEN |
| Set up comprehensive monitoring | GREEN |
| Create post-incident report | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- General troubleshooting may expose data across multiple Bedrock services and configurations

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. Rollback steps depend on the specific runbook identified during triage — follow the Phase 3 rollback of the matched runbook
2. If multiple changes were made during investigation, revert them in reverse order
3. Document all changes made during troubleshooting for audit purposes
4. Verify rollback by re-running the Phase 1 triage commands and confirming expected state

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
