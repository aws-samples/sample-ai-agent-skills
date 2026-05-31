---
title: "C1 — Category Rule Errors"
description: "Diagnose Contact Lens category rule configuration and evaluation errors"
status: active
severity: MEDIUM
triggers: ["category not matching", "rule not working", "category rule error", "no categories detected"]
owner: devops-agent
objective: "Resolve category rule configuration and evaluation issues"
context: "Category rules use pattern matching on transcripts to categorize contacts. Rules must be PUBLISHED to be evaluated. Rules can be configured for real-time or post-contact evaluation."
---

## Phase 1 — Triage

MUST:
- List rules: `aws connect list-rules --instance-id <instance-id> --publish-status PUBLISHED`
- Get rule details: `aws connect describe-rule --instance-id <instance-id> --rule-id <rule-id>`
- Verify the rule is in PUBLISHED status (draft rules are not evaluated)

SHOULD:
- Check if the rule's trigger event type matches the desired evaluation time (real-time vs post-contact)
- Verify the rule's conditions match the expected transcript patterns
- Test the rule against a known contact transcript

MAY:
- Review CloudTrail for rule creation and update events
- Check if the rule was recently modified or republished

## Phase 2 — Remediate

MUST:
- Publish draft rules: `aws connect update-rule --instance-id <instance-id> --rule-id <rule-id> --actions '{"AssignContactCategoryActions":[{}]}' --function '<function>' --name '<name>' --publish-status PUBLISHED`
- Verify rule conditions use correct syntax and pattern matching
- Ensure the rule's trigger event matches the desired evaluation time

SHOULD:
- Test rules with known phrases before deploying to production
- Use specific keyword patterns rather than overly broad matches
- Configure rule actions (assign category, send notification, create task)

MAY:
- Create separate rules for real-time and post-contact evaluation
- Set up EventBridge integration for category match notifications
- Implement rule versioning for change management

## Safety Ratings

```
safety_ratings:
  - "list-rules: GREEN — read-only rule enumeration"
  - "describe-rule: GREEN — read-only rule details"
  - "update-rule: YELLOW — modifies rule configuration"
  - "create-rule: YELLOW — creates new evaluation rule"
  - "delete-rule: RED — removes evaluation rule permanently"
```

## Escalation Conditions

- Rules fail to evaluate despite correct configuration and PUBLISHED status
- Rule pattern matching produces false positives at a high rate
- Real-time rules have unacceptable latency (more than 30 seconds)
- Rule actions (notifications, tasks) fail to execute

## Data Sensitivity

- `aws connect list-rules`: LOW — returns rule metadata
- `aws connect describe-rule`: MEDIUM — reveals rule patterns and business logic
- Category match results: MEDIUM — reveal conversation topics and issues
- Rule actions: MEDIUM — may trigger notifications or task creation

## Prohibited Actions

- NEVER delete production rules without backup or documentation
- NEVER create rules with overly broad patterns that match most contacts
- NEVER use category rules as the sole compliance monitoring mechanism
- NEVER modify rules during peak hours without testing

## Phase 3 — Rollback

1. Revert rule changes by updating to the previous configuration
2. If a rule was deleted, recreate it with the documented configuration
3. Set rules back to DRAFT status if they are causing false positives: update publish status
4. Remove EventBridge rules if they were created in error
5. Document the rule change and rollback reason

## Common Issues

- symptoms: "Rule exists but no contacts are categorized"
  diagnosis: "Rule is in DRAFT status — only PUBLISHED rules are evaluated."
  resolution: "Publish the rule by updating its publish status to PUBLISHED."

## Output Format

```yaml
root_cause: "category_rule — <specific_cause>"
evidence:
  - type: rule_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix rule configuration"
  long_term: "Implement rule testing and change management procedures"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
