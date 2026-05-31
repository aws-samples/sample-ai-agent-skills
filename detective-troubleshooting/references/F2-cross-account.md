---
title: "F2 — Cross-Account Access Issues"
description: "Diagnose Detective cross-account investigation and access problems"
status: active
severity: MEDIUM
triggers:
  - "cross-account detective"
  - "investigate other account"
  - "cross-account investigation"
  - "member account access"
owner: devops-agent
objective: "Resolve cross-account access issues for Detective investigations"
context: "Detective cross-account access is managed through the behavior graph membership model. The admin account can investigate findings from all member accounts. Member accounts can only see their own data unless granted admin access."
---

## Phase 1 — Triage

MUST:
- Verify the target account is a member: `aws detective get-members --graph-arn <graph-arn> --account-ids <target-account>`
- Check member status is ENABLED (not INVITED or VERIFICATION_IN_PROGRESS)
- Verify the investigator has admin-level access to the behavior graph
- Check data source status for the target account

SHOULD:
- Verify Organizations integration if using delegated admin
- Check if the target account's data sources are enabled
- Verify the investigation is in the correct region

MAY:
- Check CloudTrail for cross-account access events
- Verify no SCP blocking cross-account Detective operations

## Phase 2 — Remediate

MUST:
- If account not a member, invite and add to graph
- If member status not ENABLED, resolve invitation or verification
- Ensure data sources are enabled for the target account

SHOULD:
- Use Organizations auto-enable for automatic membership
- Ensure consistent data source configuration across all members

## Output Format

```yaml
root_cause: "cross_account_access — <specific_cause>"
evidence:
  - type: member_status
    content: "<target account membership status>"
  - type: datasource_status
    content: "<target account data sources>"
severity: MEDIUM
mitigation:
  immediate: "Fix membership or data source configuration"
  long_term: "Use Organizations auto-enable for consistent membership"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Cross-account access enables viewing security data from other accounts |
| Blast Radius | LOW — Access changes affect investigation capability only |
| Rollback Complexity | LOW — Membership and access can be adjusted |
| Compliance Scope | HIGH — Cross-account security investigation must be properly authorized |

## Escalation Conditions

- Escalate to security team if cross-account investigation access is blocked during an incident
- Escalate to account owners if their data needs to be investigated

## Data Sensitivity

- **Cross-account security data** is accessible to the admin account — highest sensitivity
- **Member account activity** is visible to investigators with admin access

## PROHIBITED Actions

- Do NOT investigate member account data without proper authorization
- Do NOT share cross-account investigation results outside the security team
- Do NOT remove member accounts during active cross-account investigations

## Phase 3 — Rollback

- Restore member account enrollment if it was accidentally removed
- Verify data sources are enabled for the target account
- Wait for data to re-populate if the account was recently re-added

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
  - "NEVER suggest disabling Detective to reduce costs"
  - "NEVER suggest ignoring investigation findings"
  - "NEVER expose investigation details publicly"
