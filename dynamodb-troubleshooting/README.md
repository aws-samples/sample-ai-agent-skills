# DynamoDB Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon DynamoDB problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for DynamoDB — throttling analysis, partition key design evaluation, GSI capacity issues, Streams processing, TTL behavior, global table replication, transaction failures, DAX caching, and capacity planning.

### Activate When

- Read or write throttling (ProvisionedThroughputExceededException)
- Hot partition issues
- GSI backpressure throttling base table
- Query or scan performance problems
- Auto-scaling not responding fast enough
- DynamoDB Streams processing failures
- TTL not expiring items as expected
- Global table replication lag or conflicts
- Transaction failures or conflicts
- DAX cache inconsistency
- Table stuck in CREATING/UPDATING state

---

## Runbook Library (32 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Throttling** | A1–A4 | Read/write throttling, GSI backpressure, on-demand |
| **B — Partition & Key Design** | B1–B3 | Hot partitions, key design, adaptive capacity |
| **C — Query & Scan** | C1–C3 | Query performance, scan optimization, pagination |
| **D — Capacity** | D1–D3 | Auto-scaling, provisioned vs on-demand, burst |
| **E — Indexes** | E1–E3 | GSI creation, LSI limitations, projections |
| **F — Streams** | F1–F3 | Processing failures, Lambda triggers, consumer lag |
| **G — TTL & Lifecycle** | G1–G2 | TTL expiry, filter expressions |
| **H — Global Tables** | H1–H3 | Replication lag, conflicts, failover |
| **I — Transactions** | I1–I2 | Transaction failures, idempotency |
| **J — DAX** | J1–J2 | Cache issues, consistency |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## License

MIT-0
