---
title: "I1 — Transaction Failures"
description: "Diagnose TransactionCanceledException, cancellation reasons, and idempotency tokens"
status: active
severity: HIGH
triggers:
  - "TransactionCanceledException"
  - "transaction failed"
  - "transaction conflict"
  - "transaction cancelled"
  - "idempotency token"
owner: devops-agent
objective: "Identify and resolve DynamoDB transaction failures"
context: "DynamoDB transactions provide ACID guarantees across up to 100 items in a single account and region. TransactionCanceledException includes CancellationReasons for each item. Common reasons: ConditionalCheckFailed, TransactionConflict (another transaction on the same item), ValidationError, ItemCollectionSizeLimitExceeded. Transactions consume 2x the capacity of non-transactional operations. ClientRequestToken provides idempotency for 10 minutes."
---

## Phase 1 — Triage

MUST:
- Check the TransactionCanceledException CancellationReasons in application logs
- Check for TransactionConflict errors: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name TransactionConflict --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Verify the transaction request is valid (max 100 items, single account, single region)
- Check consumed capacity for transactions: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Check if conditional expressions in the transaction are failing
- Review if multiple transactions target the same items (TransactionConflict)
- Check if the transaction exceeds the 100-item limit
- Verify all items are in the same AWS account and region

MAY:
- Check if the transaction includes items from tables with different billing modes
- Review if the transaction uses idempotency tokens (ClientRequestToken)
- Analyze transaction frequency to detect contention patterns

## Phase 2 — Remediate

MUST:
- Parse CancellationReasons to identify which item and why the transaction failed
- For ConditionalCheckFailed: review and fix the condition expression
- For TransactionConflict: implement retry with exponential backoff
- For ValidationError: fix the request format (item size, attribute types, etc.)

SHOULD:
- Use ClientRequestToken for idempotent retries: include a unique token per transaction
- Reduce transaction scope to minimize conflict probability
- Separate read and write operations — use TransactGetItems for reads

MAY:
- Implement optimistic locking with version numbers instead of transactions where possible
- Use conditional writes instead of transactions for single-item operations
- Design data models to minimize cross-item transaction needs

## Common Issues

- symptoms: "TransactionCanceledException with TransactionConflict reason"
  diagnosis: "Two transactions are modifying the same item simultaneously. DynamoDB cancels one to maintain isolation."
  resolution: "Retry with exponential backoff. Reduce contention by designing items to be updated by fewer concurrent transactions."

- symptoms: "TransactionCanceledException with ConditionalCheckFailed"
  diagnosis: "A ConditionExpression in the transaction evaluated to false. The item's current state doesn't match the expected state."
  resolution: "Read the current item state, update the condition, and retry. This is expected behavior for optimistic concurrency."

- symptoms: "Transaction succeeds but data appears duplicated"
  diagnosis: "Transaction was retried without ClientRequestToken, causing duplicate execution."
  resolution: "Always use ClientRequestToken for TransactWriteItems. Use the same token for retries of the same transaction."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix ConditionExpression | GREEN | Application-level bug fix; no infrastructure change |
| Implement retry with exponential backoff | GREEN | Application-level resilience |
| Add ClientRequestToken | GREEN | Enables idempotent retries; no data risk |
| Reduce transaction scope | GREEN | Application-level optimization |
| Use conditional writes instead of transactions | GREEN | Simpler alternative; application-level change |

## Escalation Conditions

- TransactionConflict rate exceeding 10% of transaction attempts
- Transaction failures causing data inconsistency in production
- Duplicate transaction execution detected (missing idempotency)
- Transaction failures blocking critical business operations
- ItemCollectionSizeLimitExceeded within a transaction on a table with LSIs

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| Application logs (CancellationReasons) | HIGH | May contain item keys and condition details |
| `get-metric-statistics` (TransactionConflict) | LOW | Operational metrics only |
| `get-metric-statistics` (consumed capacity) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest deleting a table to fix transaction failures
- NEVER suggest disabling encryption to improve transaction throughput
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER retry a transaction without using the same ClientRequestToken

## Phase 3 — Rollback

If transaction changes cause issues:
1. Revert application code to previous transaction implementation
2. If transaction scope was reduced and split into multiple transactions, revert to original atomic transaction
3. If conditional writes replaced transactions, revert if atomicity across items is required
4. If ClientRequestToken was implemented incorrectly, fix token generation logic
5. Verify data consistency after rollback — check for partial updates from split transactions

## Output Format

```yaml
root_cause: "transaction_failure — <specific_cause>"
evidence:
  - type: cancellation_reasons
    content: "<CancellationReasons from exception>"
  - type: transaction_conflict_metric
    content: "<TransactionConflict count>"
  - type: transaction_details
    content: "<items and operations in the transaction>"
severity: HIGH
transaction_analysis:
  failure_reason: "ConditionalCheckFailed | TransactionConflict | ValidationError | other"
  affected_items: "<list of items>"
  idempotency_token_used: "yes | no"
  retry_count: "<count>"
  concurrent_transactions: "yes | no"
mitigation:
  immediate: "Fix the specific cancellation reason and retry with backoff"
  long_term: "Implement idempotency tokens and reduce transaction contention"
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
