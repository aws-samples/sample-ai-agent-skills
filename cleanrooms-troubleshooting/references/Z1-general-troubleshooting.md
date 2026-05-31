---
title: "Z1 — General Clean Rooms Troubleshooting (Catch-All)"
description: "Fallback SOP for Clean Rooms issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Clean Rooms issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List collaborations: `aws cleanrooms list-collaborations`
- List memberships: `aws cleanrooms list-memberships`
- If specific collaboration, get details: `aws cleanrooms get-collaboration --collaboration-identifier <id>`
- List configured tables: `aws cleanrooms list-configured-tables`
- Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cleanrooms.amazonaws.com --max-results 10`

SHOULD:
- Verify IAM permissions
- Check configured table associations
- Verify analysis rules are configured

MAY:
- Check AWS Health Dashboard for Clean Rooms service events
- Check Clean Rooms ML resources: `aws cleanrooms-ml list-training-datasets`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Collaboration creation issues → Collaboration (A1)
- Membership problems → Collaboration (A2)
- Configured table errors → Tables (B1)
- Analysis rule issues → Tables (B2)
- Query execution failures → Queries (C1)
- Output configuration problems → Queries (C2)
- Differential privacy issues → Privacy (D1)
- Cryptographic computing problems → Privacy (D2)
- ML modeling failures → ML (E1)
- Lookalike segment issues → ML (E2)
- IAM permission errors → Security (F1)
- Data access problems → Security (F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all collaboration and membership configurations
- Review CloudTrail for recent Clean Rooms changes
- Verify all prerequisite services (Glue, S3) are accessible
- Test basic operations: list-collaborations, list-configured-tables

SHOULD:
- Compare with known-good configuration
- Check AWS Health Dashboard

MAY:
- Check for regional service outages
- Review Clean Rooms console for visual indicators

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<collaboration|tables|queries|privacy|ml|security|unknown>"
investigation_path: "collaboration status → membership → tables → <domain_classification>"
evidence:
  - type: collaboration_config
    content: "<collaboration configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Check all collaboration and membership configurations | GREEN |
| Review CloudTrail for recent changes | GREEN |
| Verify prerequisite services are accessible | GREEN |
| Test basic operations | GREEN |
| Compare with known-good configuration | GREEN |

## Escalation Conditions

- Issue cannot be classified into any specific failure domain
- Fix requires modifying production collaboration or privacy configurations

## Data Sensitivity

- **HIGH**: general troubleshooting may expose collaboration structures, membership details, and data access patterns
- Investigation may reveal sensitive configuration across multiple Clean Rooms components

## Prohibited Actions

- NEVER suggest disabling privacy controls
- NEVER suggest exporting raw data from collaborations

## Phase 3 — Rollback

1. If any configuration was changed during investigation, restore previous settings
2. If permissions were modified, revert to previous IAM policies
3. Document all changes made during investigation for audit purposes
4. Verify rollback by confirming all Clean Rooms resources match pre-investigation state

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
