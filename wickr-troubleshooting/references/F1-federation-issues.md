---
title: "F1 — Federation Issues"
description: "Diagnose cross-network federation problems"
status: active
severity: MEDIUM
triggers: ["federation failed", "cannot message external network", "federation request pending", "cross-network error"]
owner: devops-agent
objective: "Resolve federation configuration and cross-network communication issues"
context: "Federation allows messaging between separate Wickr networks. Both network admins must configure and approve federation."
---

## Phase 1 — Triage

MUST:
- Verify local network is active: `aws wickr get-network --network-id <id>`
- Check federation configuration in admin console
- Review CloudTrail for federation events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

SHOULD:
- Verify federation request status (pending, approved, rejected)
- Confirm the remote network has also configured federation
- Check if federation was previously working and recently broke

MAY:
- Contact the remote network administrator to verify their configuration
- Test federation with a simple message between networks

## Phase 2 — Remediate

MUST:
- Ensure both network administrators have approved the federation
- Verify federation configuration matches on both sides
- Confirm both networks are in active/healthy status

SHOULD:
- Re-send federation request if previous request expired
- Verify user accounts exist on both networks for cross-network messaging
- Document federation agreements between organizations

MAY:
- Set up federation monitoring for connectivity status
- Configure federation policies (allowed users, message types)

## Safety Ratings

```
safety_ratings:
  - "wickr get-network: GREEN — read-only network details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "federation configuration: YELLOW — enables cross-network communication"
  - "federation approval: YELLOW — grants external network access"
```

## Escalation Conditions

- Federation breaks between previously connected networks
- Federation request rejected by remote network without explanation
- Cross-network messages delivered to wrong recipients
- Federation configuration causes local network instability

## Data Sensitivity

- `aws wickr get-network`: LOW — returns network metadata
- Federation configuration: MEDIUM — reveals cross-organization relationships
- Cross-network message metadata: MEDIUM — reveals inter-organization communication patterns
- Federation agreements: MEDIUM — business relationship information

## Prohibited Actions

- NEVER approve federation requests without verifying the remote network identity
- NEVER enable federation with unknown or unverified networks
- NEVER share local network configuration details with unverified external parties
- NEVER bypass federation approval process

## Phase 3 — Rollback

1. If federation was enabled in error, disable it in the admin console
2. If federation configuration was modified, restore original settings
3. Verify local network users are unaffected by federation changes
4. Confirm no unauthorized cross-network communication occurred
5. Document federation changes and notify both network administrators

## Common Issues

- symptoms: "Federation request shows 'pending' for extended period"
  diagnosis: "Remote network administrator has not approved the federation request."
  resolution: "Contact the remote network administrator to approve the federation request."

- symptoms: "Federation was working but cross-network messages stopped"
  diagnosis: "One network may have disabled federation or network status changed."
  resolution: "Verify both networks are active. Check federation status on both sides."

## Output Format

```yaml
root_cause: "federation — <specific_cause>"
evidence:
  - type: federation_status
    content: "<status details>"
severity: MEDIUM
mitigation:
  immediate: "Restore federation connectivity"
  long_term: "Implement federation monitoring and documentation"
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
