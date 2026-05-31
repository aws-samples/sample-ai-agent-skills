---
title: "B2 — Peering Issues"
description: "Diagnose core network peering creation, acceptance, and connectivity failures"
status: active
severity: HIGH
triggers: ["peering failed", "peering stuck", "peering not working", "core network peering error", "cross-region peering"]
owner: devops-agent
objective: "Resolve core network peering establishment and connectivity issues"
context: "Core network peering connects two core networks for inter-network routing. Peering requires mutual acceptance and optional RAM sharing for cross-account scenarios."
---

## Phase 1 — Triage

MUST:
- List peerings: `aws networkmanager list-peerings --core-network-id <id>`
- Get peering details: `aws networkmanager get-core-network --core-network-id <id>`
- Verify both core networks exist and are in AVAILABLE state

SHOULD:
- Check if peering is cross-account and verify RAM sharing: `aws ram get-resource-shares --resource-owner SELF`
- Review CloudTrail for CreatePeering errors
- Verify both core network policies allow peering

MAY:
- Check route tables on both sides for peering routes
- Verify no overlapping CIDR blocks between peered networks

## Phase 2 — Remediate

MUST:
- For stuck CREATING peering: ensure the remote core network owner accepts the peering
- For cross-account peering: set up RAM resource sharing for the core network
- Accept the peering on the remote side: `aws networkmanager accept-attachment --attachment-id <peering-attachment-id>`

SHOULD:
- Verify segment-actions in both core network policies allow traffic flow across the peering
- Update routing policies to include peered network prefixes

MAY:
- Set up CloudWatch alarms for peering state changes
- Document peering topology for network team reference

## Common Issues

- symptoms: "Peering stuck in CREATING state for more than 30 minutes"
  diagnosis: "Remote core network owner has not accepted the peering request."
  resolution: "Contact the remote core network owner to accept the peering. For cross-account, verify RAM sharing is configured."

- symptoms: "Peering AVAILABLE but no traffic flowing"
  diagnosis: "Core network policies on one or both sides do not have segment-actions allowing cross-network routing."
  resolution: "Update core network policies on both sides to include appropriate segment-actions for peered traffic."

## Safety Ratings

```
safety_ratings:
  - "list-peerings: GREEN — read-only enumeration"
  - "create-peering: YELLOW — initiates peering request, no traffic impact until accepted"
  - "accept-attachment (peering): YELLOW — enables cross-network routing"
  - "delete-peering: RED — removes inter-network connectivity"
```

## Escalation Conditions

- Peering creation fails with internal errors on either side
- Cross-account RAM sharing requires organizational policy changes
- Peering acceptance causes unexpected route propagation affecting production traffic
- Overlapping CIDR blocks between peered networks causing routing conflicts

## Data Sensitivity

- `aws networkmanager list-peerings`: LOW — returns peering metadata
- `aws ram get-resource-shares`: MEDIUM — reveals cross-account sharing configuration
- `aws networkmanager get-core-network-policy`: MEDIUM — reveals network architecture

## Prohibited Actions

- NEVER delete a production peering without confirming alternative connectivity exists
- NEVER accept peering requests from unknown or unverified accounts
- NEVER modify peering-related segment-actions without understanding the routing implications on both sides
- NEVER assume peering provides transitive routing between three or more core networks

## Phase 3 — Rollback

1. If peering was created in error, delete it: `aws networkmanager delete-peering --peering-id <id>`
2. If peering was accepted in error, delete the peering to remove connectivity
3. Remove RAM resource shares if cross-account peering is no longer needed
4. Verify routes on both core networks no longer include peered prefixes
5. Document the rollback and notify both network teams

## Output Format

```yaml
root_cause: "peering_issue — <specific_cause>"
evidence:
  - type: peering_state
    content: "<peering details and error>"
severity: HIGH
mitigation:
  immediate: "Fix peering configuration"
  long_term: "Implement peering automation with mutual acceptance workflows"
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
