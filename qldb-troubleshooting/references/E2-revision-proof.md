---
title: "E2 — Revision Proof Issues"
description: "Diagnose individual revision proof failures"
status: active
severity: MEDIUM
triggers: ["proof invalid", "revision not found", "block address wrong"]
owner: devops-agent
objective: "Identify and resolve revision proof issues"
---

## Phase 1 — Triage

MUST:
- Verify document ID exists
- Check block address format (Ion text)
- Verify digest tip address is valid
- Check proof response for completeness

SHOULD:
- Verify document was committed before the digest tip
- Check if document was in the specified block

## Phase 2 — Remediate

MUST:
- Use correct block address from document metadata
- Use valid digest tip address
- Verify document ID matches

## Output Format

```yaml
root_cause: "revision_proof — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — revision proofs involve ledger data verification
- Service Disruption Risk: LOW — proof verification does not affect ledger operations
- Compliance Impact: HIGH — revision proofs are critical for audit and compliance

## Escalation Conditions
- Revision proof verification fails
- Proof generation returns errors
- Proof chain integrity compromised
- Proof verification inconsistent between requests
- Proof generation latency exceeds thresholds

## Data Sensitivity
- Revision proof data — CONFIDENTIAL
- Document revision history — CONFIDENTIAL
- Proof verification results — INTERNAL
- Ledger digest data — CONFIDENTIAL
- Never expose proof data to unauthorized parties

## Prohibited Actions
- Do NOT ignore proof verification failures
- Do NOT bypass proof verification for compliance checks
- Do NOT share proof data with unauthorized parties
- Do NOT modify ledger data to fix proof failures
- Do NOT disable proof generation
- Do NOT accept unverified proofs as valid

## Phase 3 — Rollback
- Re-generate proofs if verification fails
- Verify proof chain integrity
- Restore proof verification configurations
- Document proof verification issues
- Escalate to AWS Support if proof integrity is compromised
- Update proof verification procedures

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
