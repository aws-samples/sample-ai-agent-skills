---
title: "B2 — Entity Profile Issues"
description: "Diagnose Detective entity profile problems including missing profiles and incomplete data"
status: active
severity: MEDIUM
triggers:
  - "entity profile"
  - "missing profile"
  - "entity not found"
  - "profile incomplete"
  - "detective entity"
owner: devops-agent
objective: "Resolve entity profile issues and ensure complete entity visibility"
context: "Detective builds entity profiles for IAM users, roles, IP addresses, EC2 instances, and other entities. Profiles show behavioral baselines and activity patterns. Missing profiles indicate data source or membership issues."
---

## Phase 1 — Triage

MUST:
- Verify the entity's account is a member of the behavior graph
- Check data source status for the account: `aws detective batch-get-graph-member-datasources --graph-arn <graph-arn> --account-ids <account-id>`
- Verify sufficient time has passed for profile building (24+ hours)
- Check member status: `aws detective get-members --graph-arn <graph-arn> --account-ids <account-id>`

SHOULD:
- Verify the entity type is supported (IAM user, role, IP, EC2, S3 bucket, EKS)
- Check if the entity has recent activity in CloudTrail or VPC Flow Logs
- Verify the correct region is being queried

MAY:
- Check if the entity was recently created (new entities need activity history)
- Verify no data ingestion issues for the account

## Phase 2 — Remediate

MUST:
- If account not a member, invite and add to graph
- If data sources missing, enable required packages
- Wait 24 hours for profile data to populate

SHOULD:
- Ensure CloudTrail and VPC Flow Logs data sources are enabled for complete profiles
- Verify the entity has recent activity

## Output Format

```yaml
root_cause: "entity_profile — <specific_cause>"
evidence:
  - type: member_status
    content: "<member account status>"
  - type: datasource_status
    content: "<data source status>"
severity: MEDIUM
mitigation:
  immediate: "Fix membership or data source configuration"
  long_term: "Ensure all accounts have complete data source coverage"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Entity profiles contain behavioral patterns of users and resources |
| Blast Radius | LOW — Profile issues affect investigation quality, not data |
| Rollback Complexity | LOW — Profiles rebuild automatically with data |
| Compliance Scope | MEDIUM — Complete entity profiles support thorough investigations |

## Escalation Conditions

- Escalate to security team if entity profiles are missing for accounts under investigation
- Escalate to operations if data source issues prevent profile building

## Data Sensitivity

- **Entity profiles** reveal behavioral patterns of IAM users, roles, and resources
- **Activity baselines** contain historical behavior data — security-sensitive

## PROHIBITED Actions

- Do NOT share entity profile data outside the authorized security team
- Do NOT use entity profiles for purposes other than security investigation

## Phase 3 — Rollback

- No rollback needed — profiles rebuild automatically from ingested data
- If profiles are incomplete, verify data sources are enabled and wait for data accumulation

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
