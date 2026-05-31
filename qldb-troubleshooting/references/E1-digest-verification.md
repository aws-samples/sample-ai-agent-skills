---
title: "E1 — Digest Verification Issues"
description: "Diagnose ledger digest and verification failures"
status: active
severity: HIGH
triggers: ["verification failed", "digest mismatch", "integrity error"]
owner: devops-agent
objective: "Identify and resolve digest verification failures"
context: "Verification fails when: digest stale, hash computation wrong, or proof chain incomplete."
---

## Phase 1 — Triage

MUST:
- Get current digest: `aws qldb get-digest --name <ledger-name>`
- Get revision with proof: `aws qldb get-revision --name <ledger> --block-address '{"IonText":"<addr>"}' --document-id <id> --digest-tip-address '{"IonText":"<tip>"}'`
- Verify hash computation uses SHA-256
- Check proof chain completeness

SHOULD:
- Use QLDB verification library for hash computation
- Verify digest tip address matches the digest used
- Check if document was modified after digest was taken

## Phase 2 — Remediate

MUST:
- Get fresh digest before verification
- Use correct hash computation algorithm (SHA-256)
- Ensure proof chain is complete

SHOULD:
- Implement automated verification workflows
- Store digests externally for independent verification

## Output Format

```yaml
root_cause: "digest_verification — <specific_cause>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — digest verification is a validation activity
- Service Disruption Risk: LOW — verification does not affect ledger operations
- Compliance Impact: HIGH — digest verification is critical for data integrity compliance

## Escalation Conditions
- Digest verification fails indicating potential data tampering
- Digest generation returns errors
- Verification inconsistencies between requests
- Digest chain integrity compromised
- Verification latency exceeds thresholds

## Data Sensitivity
- Ledger digest data — CONFIDENTIAL
- Verification results — INTERNAL
- Hash chain data — CONFIDENTIAL
- Verification configurations — INTERNAL
- Never expose digest data to unauthorized parties

## Prohibited Actions
- Do NOT ignore digest verification failures
- Do NOT bypass verification for compliance checks
- Do NOT share digest data with unauthorized parties
- Do NOT modify ledger data to fix verification failures
- Do NOT disable digest generation
- Do NOT accept unverified digests as valid

## Phase 3 — Rollback
- Re-generate digests if verification fails
- Verify hash chain integrity
- Restore verification configurations
- Escalate to AWS Support if digest integrity is compromised
- Document verification issues and compliance impact
- Update verification procedures

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
