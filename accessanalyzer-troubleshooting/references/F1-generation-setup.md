---
title: "F1 — Policy Generation Setup"
description: "Diagnose policy generation setup issues"
status: active
severity: MEDIUM
triggers: ["policy generation failed", "cannot generate policy", "generation setup"]
owner: devops-agent
objective: "Resolve policy generation setup issues"
context: "Policy generation creates least-privilege policies from CloudTrail activity. Requires CloudTrail logging and IAM role activity."
---

## Phase 1 — Triage

MUST:
- Verify CloudTrail is enabled and logging: `aws cloudtrail describe-trails`
- Check CloudTrail trail covers the target region
- Verify IAM role/user has activity in CloudTrail

SHOULD:
- Check CloudTrail data retention (90+ days recommended)
- Verify S3 bucket for CloudTrail logs is accessible

MAY:
- Check generation job status: `aws accessanalyzer list-policy-generations`

## Phase 2 — Remediate

MUST:
- Enable CloudTrail if not enabled
- Ensure sufficient activity history (90+ days)
- Start generation: `aws accessanalyzer start-policy-generation --policy-generation-details '{"principalArn":"<role-arn>"}'`

SHOULD:
- Wait for generation to complete (can take hours)

MAY:
- Use CloudTrail Lake for longer retention

## Common Issues

- symptoms: "Policy generation returns empty policy"
  diagnosis: "No CloudTrail activity for the specified principal."
  resolution: "Verify principal has activity in CloudTrail. Ensure sufficient history."

## Safety Ratings

```
safety_ratings:
  - "describe-trails: GREEN — read-only CloudTrail check"
  - "list-policy-generations: GREEN — read-only status check"
  - "start-policy-generation: YELLOW — initiates background job consuming CloudTrail data"
  - "enable CloudTrail: YELLOW — creates logging infrastructure with ongoing cost"
```

## Escalation Conditions

- CloudTrail is not enabled in the account, blocking all policy generation
- Policy generation returns empty results for roles known to be active
- CloudTrail data retention is insufficient for comprehensive policy generation
- Generation job fails repeatedly with internal service errors

## Data Sensitivity

- `aws cloudtrail describe-trails`: LOW — returns trail configuration metadata
- `aws accessanalyzer start-policy-generation`: MEDIUM — processes CloudTrail activity data for a principal
- Generated policies: HIGH — reveal the minimum permissions a principal actually uses

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER use generated policies as-is in production without review and supplementation
- NEVER disable CloudTrail to troubleshoot generation issues

## Phase 3 — Rollback

1. If a policy generation job was started in error, it will complete on its own — no cancellation needed
2. If CloudTrail was enabled for generation purposes and is no longer needed, disable it: `aws cloudtrail stop-logging --name <trail-name>` (caution: affects audit logging)
3. Delete generated policy results if they contain sensitive permission data: clean up via console
4. Verify CloudTrail logging is restored to its original state
5. Document any changes to CloudTrail configuration for the security team

## Output Format

```yaml
root_cause: "generation_setup — <specific_cause>"
evidence:
  - type: generation_config
    content: "<CloudTrail and generation status>"
severity: MEDIUM
mitigation:
  immediate: "Fix CloudTrail configuration"
  long_term: "Maintain CloudTrail logging for policy generation"
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
