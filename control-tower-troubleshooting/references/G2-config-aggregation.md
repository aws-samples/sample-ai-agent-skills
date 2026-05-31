---
title: "G2 — Config Aggregation Issues"
description: "Diagnose issues with AWS Config aggregation across Control Tower accounts"
status: active
severity: MEDIUM
triggers:
  - "Config not aggregating"
  - "Compliance data missing"
  - "Config aggregator error"
  - "Config recorder issues"
owner: devops-agent
objective: "Restore Config aggregation across all governed accounts"
context: "Control Tower deploys AWS Config recorders in all governed accounts and aggregates compliance data to the Audit account. Issues arise from Config recorder failures, aggregator authorization problems, or StackSet deployment failures."
---

## Phase 1 — Triage

MUST:
- Check Config aggregator in the Audit account: `aws configservice describe-configuration-aggregators`
- Check aggregator authorization status: `aws configservice describe-pending-aggregation-requests`
- Verify Config recorders are running in member accounts
- Check for failed Config-related StackSet instances

SHOULD:
- Verify the Config delivery channel is working in member accounts
- Check if new accounts have Config properly deployed
- Verify the aggregator covers all governed regions

MAY:
- Check CloudTrail for Config API failures
- Review Config recorder configuration in a sample member account

## Phase 2 — Remediate

MUST:
- If Config recorder stopped: re-register the OU to redeploy
- If aggregator authorization missing: re-register the OU
- If StackSet failed: fix the failed stack instances

SHOULD:
- Verify compliance data appears in the Audit account after fix
- Check all governed regions for Config coverage
- Do NOT manually configure Config in CT-governed accounts

MAY:
- Set up monitoring for Config recorder status
- Implement alerts for aggregation gaps

## Common Issues

- symptoms: "Compliance data missing for some accounts"
  diagnosis: "Config recorder not running or aggregator not authorized for those accounts."
  resolution: "Re-register the OU containing the affected accounts."

- symptoms: "Config recorder stopped in member account"
  diagnosis: "Someone stopped the Config recorder or deleted the delivery channel."
  resolution: "Re-register the OU. Do not manually restart — let CT manage it."

## Safety Ratings

```
safety_ratings:
  - "describe-configuration-aggregators: GREEN — read-only aggregator config"
  - "describe-pending-aggregation-requests: GREEN — read-only authorization status"
  - "re-register OU to restore Config deployment: YELLOW — recoverable, resets CT baselines"
  - "manually configure Config in CT-governed account: RED — causes drift, conflicts with CT management"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Config recorder stopped across multiple accounts
- Aggregator authorization missing for many accounts
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Config compliance data (resource configurations and compliance status)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Do NOT manually restart Config recorders in CT-governed accounts — use OU re-registration
- Document which accounts have Config issues before starting remediation
- If compliance data gaps exist, document the time range for audit purposes

## Output Format

```yaml
root_cause: "config_aggregation — <specific issue>"
evidence:
  - type: config_aggregator
    content: "<aggregation status>"
severity: MEDIUM
mitigation:
  immediate: "Re-register OU to restore Config deployment"
  long_term: "Monitor Config recorder status across accounts"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
