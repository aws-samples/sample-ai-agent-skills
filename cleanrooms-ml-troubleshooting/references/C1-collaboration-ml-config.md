---
title: "C1 — Collaboration ML Configuration"
description: "Diagnose collaboration ML configuration and setup issues"
status: active
severity: HIGH
triggers: ["collaboration ML config", "ML not available", "collaboration setup", "ML configuration error"]
owner: devops-agent
objective: "Resolve collaboration ML configuration issues and enable ML features"
context: "Clean Rooms ML features must be explicitly configured within a collaboration. All members must accept ML terms. Configuration defines privacy parameters and allowed operations."
---

## Phase 1 — Triage

MUST:
- List collaborations: `aws cleanrooms list-collaborations --query 'collaborationList[*].{Name:name,Id:id,Status:status,MemberStatus:memberStatus}'`
- Get collaboration details: `aws cleanrooms get-collaboration --collaboration-identifier <id>`
- Check ML configuration: `aws cleanrooms list-members --collaboration-identifier <id>`

SHOULD:
- Verify all members have accepted the collaboration
- Check if ML-specific IAM permissions are in place for the collaboration role

MAY:
- Review CloudTrail for collaboration update events
- Check if the collaboration was created with ML capabilities enabled

## Phase 2 — Remediate

MUST:
- Ensure collaboration was created with ML capabilities enabled
- Verify all required members have ACTIVE membership status
- Configure ML settings with proper privacy parameters and output rules

SHOULD:
- Validate that the collaboration creator has cleanrooms:UpdateCollaboration permissions
- Ensure service roles have both cleanrooms and cleanroomsml permissions

MAY:
- Recreate the collaboration with ML capabilities if not originally enabled
- Update member roles to include ML-specific permissions

## Common Issues

- symptoms: "ML features not available in collaboration"
  diagnosis: "Collaboration was created without ML capabilities enabled."
  resolution: "ML capabilities must be enabled at collaboration creation time. Create a new collaboration with ML enabled."

- symptoms: "Member cannot access ML features"
  diagnosis: "Member has not accepted ML terms or lacks ML-specific permissions."
  resolution: "Member must accept ML configuration. Verify member IAM role includes cleanroomsml:* permissions."

## Safety Ratings

```
safety_ratings:
  - "list-collaborations: GREEN — read-only enumeration"
  - "get-collaboration: GREEN — read-only detail retrieval"
  - "update-collaboration: YELLOW — modifies collaboration settings"
  - "delete-collaboration: RED — permanently removes collaboration and all associated resources"
```

## Escalation Conditions

- Collaboration ML configuration fails with internal service errors
- Cross-account collaboration requires coordination with external organizations
- ML capabilities cannot be added to existing collaboration (requires recreation)
- Privacy parameter disagreements between collaboration members

## Data Sensitivity

- `aws cleanrooms list-collaborations`: LOW — returns collaboration metadata
- `aws cleanrooms get-collaboration`: MEDIUM — reveals member account IDs and configuration
- `aws cleanrooms list-members`: MEDIUM — reveals member details and status
- `aws cloudtrail lookup-events`: MEDIUM — may reveal member actions and identities

## Prohibited Actions

- NEVER delete a collaboration with active ML models or audience generation jobs
- NEVER modify privacy parameters without consensus from all collaboration members
- NEVER share collaboration member account IDs with unauthorized parties
- NEVER force-accept ML terms on behalf of another collaboration member

## Phase 3 — Rollback

1. If collaboration ML settings were misconfigured, update with correct parameters: `aws cleanrooms update-collaboration --collaboration-identifier <id>`
2. If a new collaboration was created to replace a misconfigured one, ensure all members join the new collaboration before decommissioning the old one
3. Migrate any existing ML resources (training datasets, models) to the new collaboration context
4. Verify all members confirm the new configuration
5. Document the configuration change and notify all stakeholders

## Output Format

```yaml
root_cause: "collaboration_ml_config — <specific_cause>"
evidence:
  - type: collaboration_error
    content: "<error details from get-collaboration>"
severity: HIGH
mitigation:
  immediate: "Fix ML configuration or recreate collaboration"
  long_term: "Use IaC templates with ML capabilities pre-configured"
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
