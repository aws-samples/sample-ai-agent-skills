---
title: "D2 — Quorum Authentication"
description: "Diagnose and resolve CloudHSM quorum authentication (M of N) issues"
status: active
severity: HIGH
triggers:
  - "quorum authentication failed"
  - "MofN error"
  - "token signing"
  - "quorum token"
  - "approval pending"
owner: devops-agent
objective: "Resolve quorum authentication configuration and approval issues on CloudHSM"
context: "CloudHSM quorum authentication (M of N access control) requires a minimum number of CO approvals for certain HSM operations like user management. This provides an additional security layer for sensitive operations. Quorum is configured using CMU and requires CO users to register their public keys. Issues arise from misconfigured quorum values, expired tokens, missing approver keys, or insufficient approvers."
---

## Phase 1 — Triage

MUST:
- Check current quorum configuration: `getQuorum` (in CMU as CO)
- List registered MofN tokens: `listMofNTokens` (in CMU)
- Verify the number of registered CO approvers meets the quorum threshold
- Check if there are pending quorum tokens: `getMofNToken` (in CMU)
- Verify CO users can log in: `loginHSM CO <username> <password>` (in CMU)

SHOULD:
- Verify each CO's public key is registered for quorum: `listUsers` and cross-reference with quorum registrations
- Check if quorum tokens have expired (tokens have a time limit)
- Verify the quorum value (M) is not greater than the number of available COs (N)

MAY:
- Review the quorum configuration history via CloudTrail
- Check if quorum was recently modified
- Verify the token signing process is using the correct key format

## Phase 2 — Remediate

MUST:
- If quorum token expired: generate a new token and collect approvals again
- If insufficient approvers: register additional CO public keys for quorum
- If quorum value too high: reduce M value (requires current quorum approval): `setQuorum <service> <M>` (in CMU)
- If CO key not registered: register the CO's public key: `registerMofNPubKey <key-file>` (in CMU)

SHOULD:
- Ensure M value is at least 2 for production environments
- Keep N (total approvers) at least M+1 for redundancy
- Document the quorum configuration and approver list

MAY:
- Implement a quorum approval workflow with documented procedures
- Set up notifications when quorum tokens are generated
- Create backup CO accounts specifically for quorum participation

## Common Issues

- symptoms: "setQuorum or createUser fails with QUORUM_NOT_MET"
  diagnosis: "The operation requires quorum approval but insufficient approvals were collected."
  resolution: "Generate a quorum token, have M COs approve it, then retry the operation."

- symptoms: "Cannot collect enough approvals — COs unavailable"
  diagnosis: "Quorum M value is too high relative to available COs."
  resolution: "If possible, reduce M value (requires current quorum). Otherwise, wait for COs to become available."

- symptoms: "Quorum token rejected — invalid signature"
  diagnosis: "CO used the wrong private key to sign the token, or the token was corrupted."
  resolution: "Verify the CO is using the private key corresponding to their registered public key. Regenerate the token."

- symptoms: "registerMofNPubKey fails with invalid key format"
  diagnosis: "Public key file is not in the expected PEM or DER format."
  resolution: "Generate a new RSA 2048-bit key pair and register the public key in PEM format."

## Output Format

```yaml
root_cause: "Quorum authentication issue — <specific_cause>"
evidence:
  - type: quorum_config
    content: "<getQuorum output>"
  - type: token_status
    content: "<quorum token details>"
severity: HIGH
mitigation:
  immediate: "Collect required approvals or adjust quorum configuration"
  long_term: "Document quorum procedures and maintain sufficient CO approver redundancy"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (getQuorum/listMofNTokens): GREEN — read-only"
  - "Phase 2 quorum value changes: RED — modifies security controls"
  - "Phase 2 public key registration: YELLOW — adds new approvers"

## Escalation Conditions

escalation_conditions:
  - "Quorum cannot be met due to unavailable COs (operational lockout)"
  - "Quorum configuration change is needed in production"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Unauthorized quorum token generation is detected"
  - "Issue involves reducing quorum value below compliance requirements"

## Data Sensitivity

data_sensitivity:
  - command: "getQuorum"
    sensitivity: HIGH
    contains: "Quorum configuration and security thresholds"
  - command: "listMofNTokens"
    sensitivity: HIGH
    contains: "Pending approval tokens and approver information"
  - command: "CO private keys"
    sensitivity: CRITICAL
    contains: "Private keys used for quorum signing — never log or share"

## Prohibited Actions

prohibited_actions:
  - "NEVER reduce quorum to 1 in production without explicit security team approval"
  - "NEVER share CO private keys used for quorum signing"
  - "NEVER bypass quorum by deleting and recreating the quorum configuration"
  - "NEVER store quorum signing keys on the same system as the HSM client"
  - "NEVER approve quorum tokens without verifying the requested operation"

## Phase 3 — Rollback

- Before changing quorum values, document the current M and N values: `getQuorum`
- If quorum value was changed incorrectly, restore the previous value (requires quorum approval at the new level)
- If a CO public key was registered incorrectly, deregister it
- If quorum is completely locked out, the cluster may need to be restored from backup as a last resort
- Verify all quorum-protected operations work correctly after configuration changes
