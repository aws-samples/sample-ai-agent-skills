---
title: "I2 — Transaction Idempotency"
description: "Implement ClientRequestToken for safe transaction retries"
status: active
severity: MEDIUM
triggers:
  - "ClientRequestToken"
  - "transaction retry"
  - "idempotent transaction"
  - "duplicate transaction"
  - "transaction idempotency"
owner: devops-agent
objective: "Implement idempotent transactions using ClientRequestToken"
context: "DynamoDB TransactWriteItems supports ClientRequestToken for idempotency. If the same token is used within 10 minutes, DynamoDB returns the original response without re-executing the transaction. This prevents duplicate writes during retries. The token must be unique per distinct transaction and reused only for retries of the same transaction. Token is case-sensitive and 1-36 characters."
---

## Phase 1 — Triage

MUST:
- Check if the application uses ClientRequestToken in TransactWriteItems calls
- Review retry logic to verify the same token is used for retries
- Check for duplicate writes that indicate missing idempotency
- Verify token uniqueness strategy (UUID, hash of transaction content, etc.)

SHOULD:
- Check if retries generate new tokens (incorrect — should reuse the same token)
- Verify the token is within the 1-36 character limit
- Check if the 10-minute idempotency window is sufficient for the retry strategy
- Review if different transactions accidentally share the same token

MAY:
- Check application logs for duplicate transaction executions
- Review the token generation strategy for uniqueness guarantees
- Test idempotency by replaying a transaction with the same token

## Phase 2 — Remediate

MUST:
- Generate a unique ClientRequestToken per distinct transaction (use UUID v4)
- Reuse the same token when retrying the same transaction
- Include ClientRequestToken in all TransactWriteItems calls

SHOULD:
- Use a deterministic token based on transaction content for natural idempotency
- Implement retry logic that preserves the original token: store the token before the first attempt
- Log the ClientRequestToken for debugging duplicate execution issues

MAY:
- Implement a token cache to detect and prevent duplicate submissions at the application level
- Use the transaction content hash as the token for content-based idempotency
- Set up monitoring for duplicate transaction attempts

## Common Issues

- symptoms: "Duplicate writes after network timeout during transaction"
  diagnosis: "Transaction succeeded but the response was lost. Retry without ClientRequestToken caused duplicate execution."
  resolution: "Always use ClientRequestToken. On retry with the same token, DynamoDB returns the original result without re-executing."

- symptoms: "IdempotentParameterMismatch error"
  diagnosis: "Same ClientRequestToken used with different transaction parameters within the 10-minute window."
  resolution: "Each unique transaction must have a unique token. Only reuse tokens for exact retries of the same transaction."

- symptoms: "Transaction idempotency not working after 10 minutes"
  diagnosis: "ClientRequestToken idempotency window is 10 minutes. After that, the same token is treated as a new transaction."
  resolution: "Ensure retries happen within 10 minutes. For longer windows, implement application-level idempotency using a separate tracking table."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Generate unique ClientRequestToken (UUID) | GREEN | Application-level improvement; no infrastructure change |
| Reuse token on retry | GREEN | Correct idempotency pattern; no risk |
| Implement token cache | GREEN | Application-level deduplication |
| Log ClientRequestToken | GREEN | Debugging improvement; no operational impact |

## Escalation Conditions

- Duplicate transaction execution detected in production (data integrity issue)
- IdempotentParameterMismatch errors occurring frequently
- Idempotency window (10 minutes) insufficient for retry strategy
- Token generation strategy producing collisions

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| Application logs (token usage) | MEDIUM | May contain transaction identifiers |
| Application code review (retry logic) | LOW | Code structure only |

## Prohibited Actions

- NEVER suggest deleting a table to fix idempotency issues
- NEVER suggest disabling encryption as part of idempotency fixes
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER reuse a ClientRequestToken for a different transaction within the 10-minute window

## Phase 3 — Rollback

If idempotency changes cause issues:
1. Revert application code to previous transaction retry logic
2. If token generation was changed and causes IdempotentParameterMismatch, fix the generation strategy
3. If token cache was implemented and causes stale entries, clear the cache
4. Verify no duplicate transactions were executed during the transition
5. Audit affected data for duplicates introduced by incorrect idempotency implementation

## Output Format

```yaml
root_cause: "transaction_idempotency — <specific_cause>"
evidence:
  - type: token_usage
    content: "<ClientRequestToken implementation details>"
  - type: duplicate_detection
    content: "<evidence of duplicate transactions>"
  - type: retry_logic
    content: "<retry implementation details>"
severity: MEDIUM
idempotency_analysis:
  client_request_token_used: "yes | no"
  token_reused_on_retry: "yes | no"
  token_generation_strategy: "<UUID | hash | other>"
  idempotency_window_sufficient: "yes | no"
  duplicate_executions_detected: "yes | no"
mitigation:
  immediate: "Add ClientRequestToken to all TransactWriteItems calls"
  long_term: "Implement deterministic token generation and application-level idempotency"
```

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
