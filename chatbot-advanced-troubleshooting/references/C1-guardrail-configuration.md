---
title: "C1 — Guardrail Configuration"
description: "Diagnose AWS Chatbot guardrail configuration issues including policy syntax, command restrictions, and permission conflicts"
status: active
severity: MEDIUM
triggers: ["guardrail blocking", "command not allowed", "guardrail policy error", "cannot run command in chat"]
owner: devops-agent
objective: "Resolve guardrail configuration issues while maintaining security controls"
context: "Guardrails use IAM policy syntax but are evaluated by Chatbot, not IAM. They restrict commands per channel. Both guardrails AND IAM role permissions must allow an action for it to succeed."
---

## Phase 1 — Triage

MUST:
- Get channel configuration with guardrails: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>`
- Review the guardrail policy document for the channel
- Check if the blocked command matches a Deny statement in the guardrail

SHOULD:
- Compare guardrail policy with the channel IAM role permissions
- Verify the guardrail policy syntax is valid
- Check if the guardrail uses explicit Deny or implicit deny (no matching Allow)

MAY:
- Test with a minimal guardrail to isolate the blocking rule
- Review other channel configurations for comparison

## Phase 2 — Remediate

MUST:
- Update guardrail policy to allow the required commands
- Ensure guardrail policy syntax follows IAM policy format
- Verify both guardrails and IAM role allow the desired actions

SHOULD:
- Follow least-privilege in guardrail configuration
- Document guardrail policies and their intended restrictions
- Test guardrail changes in a non-production channel first

MAY:
- Create guardrail templates for different team roles
- Implement guardrail policy version control

## Safety Ratings

```
safety_ratings:
  - "describe-slack-channel-configurations: GREEN — read-only configuration"
  - "update-slack-channel-configuration: YELLOW — modifies guardrail policy"
  - "update-microsoft-teams-channel-configuration: YELLOW — modifies guardrail policy"
  - "Testing commands in chat: YELLOW — executes AWS operations"
```

## Escalation Conditions

- Guardrail changes affect security posture for multiple channels
- Required commands conflict with organizational security policies
- Guardrail policy complexity exceeds manageable levels
- Emergency operations blocked by guardrails during incidents

## Data Sensitivity

- `aws chatbot describe-slack-channel-configurations`: LOW — configuration metadata
- Guardrail policy document: LOW — contains policy rules
- Command execution results: MEDIUM — may contain AWS resource data
- Channel configuration: LOW — channel and role metadata

## Prohibited Actions

- NEVER remove all guardrails to fix command access issues
- NEVER use wildcard (*) Allow statements in guardrails without justification
- NEVER modify guardrails during active incident response without documentation
- NEVER copy guardrails between channels without reviewing channel-specific requirements

## Phase 3 — Rollback

1. If guardrails were modified incorrectly, restore previous policy
2. Verify previously working commands still function after changes
3. Check that security restrictions are still in place
4. Notify channel users about guardrail changes
5. Document the guardrail change and reason

## Common Issues

- symptoms: "Command works with AWS CLI but is blocked in chat"
  diagnosis: "The channel guardrail policy does not allow the command, even though the IAM role has permissions."
  resolution: "Update the guardrail policy to include an Allow statement for the command's AWS action."

- symptoms: "Guardrail update fails with policy validation error"
  diagnosis: "The guardrail policy has syntax errors — it must follow IAM policy JSON format."
  resolution: "Validate the policy JSON. Ensure it has Version, Statement with Effect, Action, and Resource fields."

- symptoms: "Some commands work but similar ones are blocked"
  diagnosis: "The guardrail Allow statement does not cover all required action variations."
  resolution: "Review the guardrail policy actions. Use wildcards carefully (e.g., ec2:Describe*) to cover related actions."

## Output Format

```yaml
root_cause: "guardrail_configuration — <specific_cause>"
evidence:
  - type: guardrail_error
    content: "<guardrail policy and blocked command details>"
severity: MEDIUM
mitigation:
  immediate: "Fix guardrail policy"
  long_term: "Implement guardrail templates and version control"
```

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
