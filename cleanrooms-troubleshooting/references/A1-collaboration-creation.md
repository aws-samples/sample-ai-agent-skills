---
title: "A1 — Collaboration Creation Issues"
description: "Diagnose Clean Rooms collaboration creation failures including member requirements and configuration"
status: active
severity: HIGH
triggers:
  - "create collaboration"
  - "collaboration failed"
  - "collaboration error"
  - "clean rooms setup"
owner: devops-agent
objective: "Resolve collaboration creation failures and establish correct configuration"
context: "Clean Rooms collaborations require at least two member accounts, proper IAM permissions, and valid configuration. Common issues include single-account attempts, missing permissions, invalid query compute settings, and member invitation failures."
---

## Phase 1 — Triage

MUST:
- List existing collaborations: `aws cleanrooms list-collaborations`
- Verify IAM permissions include `cleanrooms:CreateCollaboration`
- Verify at least two AWS accounts are available for the collaboration
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCollaboration --max-results 5`

SHOULD:
- Verify the query compute configuration (who pays for compute)
- Check if the creator account has the correct member abilities configured
- Verify no service quota limits are exceeded

MAY:
- Check for SCP restrictions on Clean Rooms operations
- Verify KMS key access if using custom encryption

## Phase 2 — Remediate

MUST:
- Ensure two separate AWS accounts are used
- Create collaboration with correct member abilities:
  ```
  aws cleanrooms create-collaboration --name <name> --members '[{"accountId":"<account1>","memberAbilities":["CAN_QUERY"],"displayName":"Member1"},{"accountId":"<account2>","memberAbilities":["CAN_RECEIVE_RESULTS"],"displayName":"Member2"}]' --creator-member-abilities CAN_QUERY --creator-display-name "Creator" --query-log-status ENABLED
  ```
- Fix IAM permissions if access denied

SHOULD:
- Enable query logging for audit purposes
- Configure appropriate member abilities for each participant

## Output Format

```yaml
root_cause: "collaboration_creation — <specific_cause>"
evidence:
  - type: error_message
    content: "<creation error>"
  - type: permissions
    content: "<IAM permissions check>"
severity: HIGH
mitigation:
  immediate: "Fix configuration or permissions"
  long_term: "Manage collaborations via IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure two separate AWS accounts are used | GREEN |
| Create collaboration with correct member abilities | YELLOW |
| Fix IAM permissions if access denied | YELLOW |
| Enable query logging for audit purposes | GREEN |
| Configure appropriate member abilities | GREEN |

## Escalation Conditions

- Fix requires modifying production collaboration configurations
- Fix involves changing privacy or data sharing settings across accounts

## Data Sensitivity

- **HIGH**: collaboration configurations expose member account IDs, data sharing agreements, and query capabilities
- Member abilities and query logging settings reveal data governance posture

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If collaboration was created with wrong configuration, delete and recreate: `aws cleanrooms delete-collaboration --collaboration-identifier <id>`
2. If member abilities were misconfigured, update the collaboration settings
3. If IAM permissions were broadened, revert to the previous policy
4. Verify rollback by confirming `aws cleanrooms get-collaboration` shows expected configuration

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
