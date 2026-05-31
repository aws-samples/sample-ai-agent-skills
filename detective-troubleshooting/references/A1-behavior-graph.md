---
title: "A1 — Behavior Graph Issues"
description: "Diagnose Detective behavior graph creation, configuration, and status problems"
status: active
severity: HIGH
triggers:
  - "behavior graph"
  - "create graph failed"
  - "graph not working"
  - "detective graph"
  - "enable detective"
owner: devops-agent
objective: "Resolve behavior graph issues and ensure Detective is operational"
context: "The behavior graph is the core data structure in Detective. Creation requires GuardDuty to be enabled for 48+ hours. Common issues include GuardDuty prerequisite, IAM permissions, and graph configuration."
---

## Phase 1 — Triage

MUST:
- List existing graphs: `aws detective list-graphs`
- Verify GuardDuty is enabled: `aws guardduty list-detectors`
- Check GuardDuty detector status: `aws guardduty get-detector --detector-id <id>`
- Verify IAM permissions include `detective:CreateGraph`, `detective:CreateMembers`
- Check if graph already exists in this region

SHOULD:
- Verify GuardDuty has been enabled for at least 48 hours
- Check CloudTrail for Detective API errors
- Verify the account is not already a member of another graph

MAY:
- Check service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForDetective`
- Verify no SCP blocking Detective operations

## Phase 2 — Remediate

MUST:
- If GuardDuty not enabled, enable it first: `aws guardduty create-detector --enable`
- Wait 48 hours after GuardDuty enablement before creating graph
- If permissions missing, add Detective permissions to IAM policy
- Create graph: `aws detective create-graph --tags '{"Environment":"production"}'`

SHOULD:
- Enable auto-enable for Organizations if using multi-account
- Configure data source packages after graph creation

## Output Format

```yaml
root_cause: "behavior_graph — <specific_cause>"
evidence:
  - type: graph_status
    content: "<list-graphs output>"
  - type: guardduty_status
    content: "<GuardDuty detector status>"
severity: HIGH
mitigation:
  immediate: "Fix prerequisites and create/configure graph"
  long_term: "Automate Detective enablement with Organizations integration"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Behavior graph contains security investigation data from multiple accounts |
| Blast Radius | HIGH — Graph deletion destroys all historical security investigation data |
| Rollback Complexity | HIGH — Deleted graphs cannot be recovered; data must be re-ingested over weeks |
| Compliance Scope | CRITICAL — Detective supports security incident investigation requirements |

## Escalation Conditions

- Escalate to security team if behavior graph creation fails blocking security investigations
- Escalate to management if GuardDuty prerequisite is not met (48-hour requirement)
- Escalate to AWS Support if graph gets stuck in an error state

## Data Sensitivity

- **Behavior graph data** contains security-relevant activity from all member accounts
- **Graph ARN** identifies the security investigation scope — restrict access
- **GuardDuty findings** ingested into the graph are security-sensitive

## PROHIBITED Actions

- Do NOT delete behavior graphs without security team approval — all investigation data is lost
- Do NOT create graphs in accounts without proper security governance
- Do NOT disable GuardDuty while Detective is active
- Do NOT share graph ARNs outside the security team

## Phase 3 — Rollback

- If graph was accidentally deleted, create a new graph and re-invite all members
- Note: historical data is lost — new graph starts fresh with 48-hour GuardDuty requirement
- Re-enable all data source packages on the new graph
- Document the data loss incident for security audit records

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
