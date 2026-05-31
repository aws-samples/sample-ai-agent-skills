---
title: "Z1 — General Troubleshooting"
description: "General advanced AWS Chatbot troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["chatbot issue", "chatbot not working", "general chatbot problem", "chatbot help"]
owner: devops-agent
objective: "Provide general troubleshooting guidance for advanced AWS Chatbot issues"
context: "This catch-all runbook covers general advanced Chatbot troubleshooting when the issue does not clearly map to a specific runbook. Start here and escalate to specific runbooks as the issue becomes clearer."
---

## Phase 1 — Triage

MUST:
- List all Chatbot configurations: `aws chatbot describe-slack-channel-configurations` and `aws chatbot list-microsoft-teams-channel-configurations`
- Check IAM roles associated with configurations: `aws iam get-role --role-name <role-name>`
- Review CloudTrail for recent Chatbot events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=chatbot.amazonaws.com --max-results 20`

SHOULD:
- Verify SNS topic subscriptions for each configuration
- Check if the issue is platform-specific (Slack vs Teams)
- Review recent configuration changes

MAY:
- Check AWS Health Dashboard for Chatbot service issues
- Test basic notification delivery with a simple SNS publish
- Review Chatbot service quotas

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Refer to the appropriate specific runbook (A1-F1) based on findings
- Verify the fix resolves the issue

SHOULD:
- Document the issue and resolution
- Set up monitoring for the specific failure mode
- Review related configurations for similar issues

MAY:
- Implement preventive measures
- Create runbooks for recurring issues
- Review and update Chatbot configurations based on lessons learned

## Safety Ratings

```
safety_ratings:
  - "describe-slack-channel-configurations: GREEN — read-only configuration listing"
  - "list-microsoft-teams-channel-configurations: GREEN — read-only configuration listing"
  - "iam get-role: GREEN — read-only role metadata"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook after initial triage
- Multiple Chatbot features failing simultaneously
- Chatbot service appears to be experiencing regional issues
- Issue requires AWS Support engagement

## Data Sensitivity

- `aws chatbot describe-slack-channel-configurations`: LOW — configuration metadata
- `aws chatbot list-microsoft-teams-channel-configurations`: LOW — configuration metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions
- `aws iam get-role`: LOW — returns role metadata

## Prohibited Actions

- NEVER make changes to production Chatbot configurations without understanding the root cause
- NEVER disable Chatbot features to work around issues
- NEVER ignore recurring Chatbot failures
- NEVER modify IAM permissions broadly to fix specific Chatbot issues

## Phase 3 — Rollback

1. Revert any diagnostic changes made during troubleshooting
2. Restore original Chatbot configurations if modified
3. Verify all Chatbot features function correctly after troubleshooting
4. Document findings and share with the team
5. Schedule a review if the root cause was not fully identified

## Common Issues

- symptoms: "All Chatbot notifications stopped across all channels"
  diagnosis: "IAM role change, SNS topic deletion, or Chatbot service issue."
  resolution: "Check IAM roles, SNS topics, and Chatbot service health. Review CloudTrail for recent changes."

- symptoms: "Chatbot works in Slack but not in Teams (or vice versa)"
  diagnosis: "Platform-specific configuration issue — different setup requirements."
  resolution: "Review platform-specific configuration. Slack and Teams have different integration models."

- symptoms: "Chatbot commands work but notifications do not"
  diagnosis: "Command execution and notification delivery use different paths — SNS vs direct API."
  resolution: "Check SNS topic subscriptions and notification preferences separately from command configuration."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details and diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and preventive measures"
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
